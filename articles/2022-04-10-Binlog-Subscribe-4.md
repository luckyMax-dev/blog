# Binlog 增量订阅 4：字段解析

在创建表结构时，可以为字段定义不同的数据类型，例如 bigint、varchar、date 等，接下来看看这些数据类型是如何存储在 ROWS_EVENT 包含的字节流中。

要想从 ROWS_EVENT 包含的字节流中解析字段值，需要借助 TABLE_MAP_EVENT，之前的小节介绍过，这个事件包含了一个表当时的元信息，该事件的几个特点：

- 实时性，记录的是修改时的表结构信息；
- 包含一张表定义的所有字段类型；
- 包含每个字段的元信息，例如 varchar 长度，datetime 精度，数值类精度；
- 表中有几个字段，哪些字段可以为 null，哪些字段不可以为 null；

有了这些信息，就可以从 ROWS_EVENT 的字节流中解析字段的值，以上一节创建的表为例：

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

table map：

![table_map_event](https://github.com/notayessir/blog/blob/main/images/binlog/table_map_event.png)

rowsData：

![rows_event](https://github.com/notayessir/blog/blob/main/images/binlog/rows_event.png)

columnDef 说明了这 6 个字段类型分别是 bigint、varchar、int、datetime、enum、double；

columnMetaDef 说明类型的元属性分别为，bigint[]、varchar[-3,2]、int[]、datetime[0]、enum[-9,1]、double[8]；

rowsData 字节流：

```
0, 1, 0, 0, 0, 0, 0, 0, 0, 3, 0, 108, 101, 111, 18, 0, 0, 0, -103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

在解析字段之前，如何知道这次修改的数据包含哪些字段？

答案是，根据 ROWS_EVENT 协议帧里的 nul-bitmap 字段，根据读取规则，我们能从 rowsData 字节流中读到 nul-bitmap 字段只有 1 个字节，即为 0，将其转为 2 进制 00 00 00 00，可以知道这次修改的字段全部不为空，按顺序读取即可，剩余字节为：

```
1, 0, 0, 0, 0, 0, 0, 0, 3, 0, 108, 101, 111, 18, 0, 0, 0, -103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

根据以上信息，依次解析字段。

### bigint

bigint 类型固定占用 8 个字节，所以我们从 rowsData 读取 8 个连续的字节为 [1, 0, 0, 0, 0, 0, 0]，按 least significant byte 读取，id 字段值为 1；

剩余字节为：

```
3, 0, 108, 101, 111, 18, 0, 0, 0, -103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

### varchar

varchar 类型首先按照 int\<lenenc> 规则读取字符串长度，得到长度为 3，再读取 1 个字节表示精度 0，然后读取连续的 3 个字节 [108, 101, 111]，即 name 字段的值为 leo；

剩余字节为：

```
18, 0, 0, 0, -103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

### int

int 类型固定占用 4 字节，读取连续的 4 个字节 [18, 0, 0, 0]，即 age 字段的值为 18；

剩余字节为：

```
-103, -84, -110, -11, 90, 2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

### datetime

datetime 类型在不同 MySQL 版本中的存储不同，具体介绍在官方文档 [Date and Time Type Storage Requirements](https://dev.mysql.com/doc/refman/8.0/en/storage-requirements.html#data-types-storage-reqs-date-time) 做了详细说明，这里根据 MySQL 5.6.4 之后的版本进行解析；

首先读取 5 个字节 [-103, -84, -110, -11, 90]，根据 [Date and Time Data Type Representation](https://dev.mysql.com/doc/internals/en/date-and-time-data-type-representation.html) 规则解析出时间 2022-4-9 15:21:26，再读取小数部分，此时需要借助 table map 信息 datetime[0]，说明该字段小数部分占用字节数为 0，即无小数部分，最终得到 birth_date 的值为 2022-4-9 15:21:26；

剩余字节为：

```
2, -51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

### enum

enum 类型在 MySQL 内部实际是 string 类型的一种，为了从中确定子类型，需要借助 table map 信息 enum[-9,1]，首先利用 -9 能够确定该子类型为 enum，之后利用 1 确定需要从字节流中读取多少字节，即 1 个字节，读到的字节值为 2，注意我们数据库中定义的枚举类型有 'female','male','undeifne'，2 并不是这 3 个值中之一，但可以猜到，2 其实代表的是数组索引，将枚举反转为 'undeifne','male' ,'female'，2 的索引即为 female，即 sex 的值为 female，MySQL 这么做应该是为了提高字节传输效率；

剩余字节为：

```
-51, -52, -52, -52, -52, -52, -4, 63, 49, -90, -87, 109
```

### double

double 类型固定占用 8 字节，从剩余字节流中读取 8 字节为 [-51, -52, -52, -52, -52, -52, -4, 63]，按 least significant byte 读取，height 字段值为 1.8；

最后剩余字节为 CRC32 计算得出的，用于确认字节流在传输中是否有误，更多 CRC32 介绍可以参考下面文档：

```
49, -90, -87, 109
```

关于更多的字段的解析可以参考下列的官方文档。

### 参考文档

Data Types：https://dev.mysql.com/doc/refman/8.0/en/data-types.html

Date and Time Type Storage Requirements：https://dev.mysql.com/doc/refman/8.0/en/storage-requirements.html#data-types-storage-reqs-date-time

CRC32：https://en.wikipedia.org/wiki/Cyclic_redundancy_check
