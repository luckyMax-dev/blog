这一节，我们来看看一个协议帧包含哪些内容。

### 协议包通用头部

不同功能的协议帧，都拥有一个协议头部，格式如下：

```
int<3>          payload_length		// 整个协议包 payload 的字节长度
int<1>          sequence_id				// 包序号
string<var>    	payload						// 协议包实际内容
```

需要注意 sequence_id 字段，在 Binlog Network Stream 之前交互的协议包，需要对 sequence_id 自增 1 后才能正常完成注册 slave 流程，sequence_id 作用如下图：

![seq](https://github.com/notayessir/blog/blob/main/images/binlog/seq.png)

### 两个概念

#### Binlog Network Stream

当 binlog dump 发送之后，后续的一系列事件也称为  Binlog Network Stream；

#### Binlog Version

MySQL 目前有 4 个不同格式的 binlog 日志版本，例如同一类型的事件，因为版本不同，事件所拥有的字段不同，笔者全篇以 4 版本为例记录，下表是数据库版本对应的 binlog 版本；

| Binlog version | MySQL Version         |
| -------------- | --------------------- |
| 1              | MySQL 3.23 - < 4.0.0  |
| 2              | MySQL 4.0.0 - 4.0.1   |
| 3              | MySQL 4.0.2 - < 5.0.0 |
| 4              | MySQL 5.0.0+          |

### Binlog Event header

每个 binlog 事件都会携带的事件头部，格式如下：

```
4              timestamp		// 4 字节时间戳
1              event type		// 事件类型
4              server-id		// 服务器 id
4              event-size		// 事件数量
   if binlog-version > 1:		// v4 版本以上还有以下字段
4              log pos			// 事件所在的 binlog 文件位置
2              flags
```

事件头部总共 19 个字节，其中最重要的两个字段分别是：

- event type：用来表明该事件是什么类型的事件，后续该如何解析字节流；
- Log pos：事件所在 binlog 文件的位置，方便记录已处理的进度；

下面举例几个 binlog 事件。

### binlog 事件类型与结构

#### FORMAT_DESCRIPTION_EVENT

该事件为 binlog v4 才拥有的事件，用来告诉 slave，master 所在的 MySQL 版本，slave 可以根据数据库版本做不同的逻辑，该事件通常是一个完整 binlog 文件的首个事件，格式如下：

```
2                binlog-version					// binlog 版本
string[50]       mysql-server version		// 数据库版本
4                create timestamp				// 生成时间戳
1                event header length		// 事件头部长度，即 19
string[p]        event type header lengths	
```

在 MySQL 8 的版本中，一个 FORMAT_DESCRIPTION_EVENT 的字节流如下：

```
122, 0, 0, 2, 0, 52 ,-121 ,27 , 98, 15 ,1 ,0 ,0 ,0 ,121 ,0 ,0 ,0 ,125 ,0 ,0 ,0 ,0 ,0 ,4 ,0 ,56 ,46 ,48 ,46 ,50 ,54 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,19 ,0 ,13 ,0 ,8 ,0 ,0 ,0 ,0 ,4 ,0 ,4 ,0 ,0 ,0 ,97 ,0 ,4 ,26 ,8 ,0 ,0 ,0 ,8 ,8 ,8 ,2 ,0 ,0 ,0 ,10 ,10 ,10 ,42 ,42 ,0 ,18 ,52 ,0 ,10 ,40 ,1 ,36 ,30 ,-101 ,-103
```

根据字段信息，得到如下图所示：

![format_event](https://github.com/notayessir/blog/blob/main/images/binlog/format_event.png)

#### ROTATE_EVENT

binlog 文件不可能无限大，当 binlog 文件达到切割条件时，MySQL 会在该 binlog 文件的最后加上该事件，来告诉 slave，这份 binlog 文件读取完后，接下来读取的 binlog 文件名称，格式如下：

```
if binlog-version > 1 {
8              position		// 下一份文件，首个事件的起始位置
}
string[p]      name of the next binlog // 下一份 binlog 文件名称
```

自行实现的框架中，会利用该事件获取 binlog 事件的名称，以记录接收到的 binlog 日志进度；

在 MySQL 8 的版本中，一个 ROTATE_EVENT 的字节流如下：

```
45, 0, 0, 1, 0, 0, 0, 0, 0, 4, 1, 0, 0, 0, 44, 0, 0, 0, 0, 0, 0, 0, 32, 0, 4, 0, 0, 0, 0, 0, 0, 0, 
98, 105, 110, 108, 111, 103, 46, 48, 48, 48, 48, 50, 56, -49, 8, 32, 37
```

根据字段信息，得到如下图所示：

![rotate_event](https://github.com/notayessir/blog/blob/main/images/binlog/rotate_event.png)

#### ROWS_QUERY_EVENT

该事件用于记录执行的 DML 语句，如果要开启这个功能，需要在 my.ini 文件里将 binlog_rows_query_log_events 字段设置为 1，格式如下：

```
1              length			
string.EOF     query text	// DDL 语句
```

在 MySQL 8 的版本中，一个 ROWS_QUERY_EVENT 的字节流如下：

```
92, 0, 0, 6, 0, -6, -102, 27, 98, 29, 1, 0, 0, 0, 91, 0, 0, 0, -95, 1, 0, 0, -128, 0, 67, 85, 80, 68, 65, 84, 69, 32, 96, 98, 105, 110, 108, 111, 103, 95, 100, 97, 116, 97, 96, 46, 96, 116, 95, 112, 102, 95, 109, 101, 109, 98, 101, 114, 96, 32, 83, 69, 84, 32, 96, 116, 101, 115, 116, 96, 32, 61, 32, 39, 54, 39, 32, 87, 72, 69, 82, 69, 32, 96, 105, 100, 96, 32, 61, 32, 49, 49, 99, -95, -63, 49
```

根据字段信息，得到如下图所示：

![row_query_event](https://github.com/notayessir/blog/blob/main/images/binlog/row_query_event.png)

#### XID_EVENT

与事务相关的事件，当表的引擎为 innodb 时，每条 DDL 语句都会生成该事件，该事件为 DDL 语句生成唯一的 id，格式如下：

```
8              xid	// long 类型的唯一 id
```

#### TABLE_MAP_EVENT

用于记录一个数据表里，所拥有字段、字段的类型、长度等，这也是解析表里每条数据变动的前置事件，格式如下：

```
post-header:
    if post_header_len == 6 {
  4              table id		// MySQL 为每个表生成一个唯一的 id，表结构不变的情况下，table id 不变
    } else {
  6              table id
    }
  2              flags

payload:
  1              schema name length
  string         schema name			// 数据库名
  1              [00]
  1              table name length
  string         table name				// 表名
  1              [00]
  lenenc-int     column-count			// 表所拥有的字段数量
  string.var_len [length=$column-count] column-def		// 字段类型
  lenenc-str     column-meta-def											// 每个字段类型的属性，例如 varchar 长度等，精度
  n              NULL-bitmask, length: (column-count + 8) / 7		// 哪些字段为 null，哪些字段不为 null
```

为了看清楚该协议事件的字节结构，先创建如下一张表：

```sql
CREATE TABLE `t_user` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `age` int DEFAULT NULL,
  `birth_date` datetime DEFAULT NULL,
  `sex` enum('female','male','undeifne') DEFAULT NULL,
  `height` double DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

新增一条数据：

```sql
INSERT INTO `binlog_data`.`t_user` (`name`, `age`, `birth_date`, `sex`, `height`) VALUES ('leo', 18, '2022-04-09 15:30:42', 'female', 70.56)
```

得到该事件的字节流：

```
74, 0, 0, 10, 0, 23, 54, 81, 98, 19, 1, 0, 0, 0, 73, 0, 0, 0, 88, -124, 6, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 11, 98, 105, 110, 108, 111, 103, 95, 100, 97, 116, 97, 0, 6, 116, 95, 117, 115, 101, 114, 0, 6, 8, 15, 3, 18, -2, 5, 6, -3, 2, 0, -9, 1, 8, 62, 1, 1, 0, 2, 1, 33, 20, 64, -92, 34
```

根据字段信息，得到如下图所示：

![table_map_event](https://github.com/notayessir/blog/blob/main/images/binlog/table_map_event.png)

columnCount 说明了该表有 6 个字段；

columnDef 说明了这 6 个字段类型分别是 bigint、varchar、int、datetime、enum、double；

columnMetaDef 说明类型的元属性分别为，bigint[]、varchar[-3,2]、int[]、datetime[0]、enum[-9,1]、double[8]；

nullMask 62 转换为 2 进制即为 00111110，1 代字段可以为 null，0 代表字段不可以为 null，从右到左，与表的字段对应上：

| 1      | 1    | 1          | 1    | 1    | 0    |
| ------ | ---- | ---------- | ---- | ---- | ---- |
| height | sex  | birth_date | age  | name | id   |

将这些信息解析出来之后，再来看看 ROWS_EVENT。

#### ROWS_EVENT

该事件类型下有 3 个子类型，3 个字类型格式类似，会携带某一行数据被修改前和被修改后的信息，区分是：

- UPDATE_ROWS_EVENTv2：update 语句，会包含新/旧数据；
- WRITE_ROWS_EVENTv2：insert 语句，只有新数据；
- DELETE_ROWS_EVENTv2：delete 语句，只有旧数据；

格式如下：

```
header:
  if post_header_len == 6 {
4                    table id			// 表 id
  } else {
6                    table id
  }
2                    flags
  if version == 2 {
2                    extra-data-length
string.var_len       extra-data
  }

body:
lenenc_int           number of columns		// 表所拥有的字段
string.var_len       columns-present-bitmap1, length: (num of columns+7)/8  
if UPDATE_ROWS_EVENTv1 or v2 {		
string.var_len       columns-present-bitmap2, length: (num of columns+7)/8 	// 
}

rows:
// 字节数组，使用 bitset 的有效位表示该 sql 语句修改了哪些字段；
// 1）如果为 insert，表示插入了哪些不为空的新数据字段；
// 2）如果为 delete，表示删除了哪些不为空的数据字段；
// 3）如果为 update，表示修改前不为空的数据字段；
string.var_len       nul-bitmap, length (bits set in 'columns-present-bitmap1'+7)/8
// 被修改的字段值
string.var_len       value of each field as defined in table-map	
if UPDATE_ROWS_EVENTv1 or v2 {			
// 同上，如果是 update 语句，这里代表修改后的数据字段和值
string.var_len       nul-bitmap, length (bits set in 'columns-present-bitmap2'+7)/8
string.var_len       value of each field as defined in table-map
}
  ... repeat rows until event-end
```

接着上一段，在插入一条数据之后，得到该事件的字节流：

```
68, 0, 0, 11, 0, 23, 54, 81, 98, 32, 1, 0, 0, 0, 67, 0, 0, 0, -101, -124, 6, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 2, 0, 6, -1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 3, 0, 108, 101, 111, 18, 0, 0, 0, -103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

根据字段信息，得到如下图所示：

![rows_event](https://github.com/notayessir/blog/blob/main/images/binlog/rows_event.png)

rowsData 字段即本次插入字段的值，下一节介绍如何从 rowsData 解析出对应字段的值。

### 参考文档

MySQL Packets 格式：https://dev.mysql.com/doc/internals/en/mysql-packet.html

Binlog Event 类型与格式：

- https://dev.mysql.com/doc/internals/en/binlog-event.html
- https://dev.mysql.com/doc/dev/mysql-server/latest/namespacebinary__log.html#ae2e9773ce2b58a3f7faad5390095c4db

字段类型：https://dev.mysql.com/doc/internals/en/com-query-response.html#packet-Protocol::MYSQL_TYPE_DECIMAL
