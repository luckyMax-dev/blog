# Log4j2 自定义 Appender

Log4j2 官方提供了多个开箱即用的 Appender 供开发者使用，如：

- AsyncAppender，可以显著的提高文件日志输出性能；
- SMTPAppender 可以将日志事件发送到指定邮箱；
- HttpAppender  可以将日志事件发送到指定接口；

当不满足业务需求时，还可以自定义 Appender，本文记录：

1. 自定义一个 HTTPAppender 的几个步骤；
2. Log4j2 使用到的建造者模式以及解耦设计；

### 自定义 Appender

#### 定义 Appender

假设有一个业务场景是统计 try catch 的异常信息，异常出现一定次数之后，调用一个 HTTP 接口进行邮件告警。

根据官方文档很容易实现这一功能，首先是定义 Appender：

```java
// 1、继承 AbstractAppender 类并重写 append 方法
// 2、使用 @Plugin 注解，name 的值会配置在 log4j2 的配置文件中，名称可以不与类名一致，这里 category、elementType、printObject 是固定值
@Plugin(name = "HTTPAlert", category = Core.CATEGORY_NAME, elementType = Appender.ELEMENT_TYPE, printObject = true)
public class MyHTTPAppender extends AbstractAppender {
    // 复用官方实现的 HttpAppender
    private final HttpAppender httpAppender;
	
    @Override
    public void append(LogEvent event) {
		// 输出日志前的逻辑
        httpAppender.append(event);
        // 如初日志后的逻辑
    }
}
```

实现 Appender 的工厂方法：

```java
@Plugin(name = "HTTPAlert", category = Core.CATEGORY_NAME, elementType = Appender.ELEMENT_TYPE, printObject = true)
public class MyHTTPAppender extends AbstractAppender {
    
    // 定义构造方法
    public MyHTTPAppender(String name, Filter filter, Layout<? extends Serializable> layout, boolean ignoreExceptions, Property[] properties, HttpAppender httpAppender) {
        super(name, filter, layout, ignoreExceptions, properties);
        // 引用外部构造好的 httpAppender
        this.httpAppender = httpAppender;
    }
    
    // 添加 @PluginFactory 注解，实现方法名为 createAppender 的静态方法
    @PluginFactory
    public static HTTPAlertAppender createAppender(@PluginConfiguration final org.apache.logging.log4j.core.config.Configuration config, @PluginAttribute("name") String name, @PluginAttribute("ignoreExceptions") boolean ignoreExceptions, @PluginElement("HTTPAlertLayout") Layout<?> layout){
        Filter filter = createDefaultFilter();
        HttpAppender httpAppender = buildHttpAppender(name, config, layout, ignoreExceptions);
        return new MyHTTPAppender(name, filter, layout, ignoreExceptions, null, httpAppender);
    }
    
    // 构造 HttpAppende，例如 http 接口名，调用参数，超时参数
    private static HttpAppender buildHttpAppender(String name, org.apache.logging.log4j.core.config.Configuration config, Layout<?> layout, boolean ignoreExceptions) {
        String endpoint = "http://my.domain.com/alert";
        int connectTimeoutMillis = 1000;
        int readTimeoutMillis = 2000;
        boolean verifyHostname = false;
        try {
            return HttpAppender.newBuilder()
                    .setIgnoreExceptions(ignoreExceptions)
                    .setLayout(layout)
                    .setConfiguration(config)
                    .setName(name)
                    .setUrl(new URL(endpoint))
                    .setConnectTimeoutMillis(connectTimeoutMillis)
                    .setReadTimeoutMillis(readTimeoutMillis)
                    .setVerifyHostname(verifyHostname)
                    .build();
        }catch (Exception e){
            System.out.println("ex:" + e);
            e.printStackTrace();
            return null;
        }
    }
    
    // 创建日志过滤器，这里只拦截 error 级别的日志
    private static Filter createDefaultFilter() {
        Level minLevel = Level.ERROR;
        Level maxLevel = Level.ERROR;
        Filter.Result match = Filter.Result.NEUTRAL;
        Filter.Result mismatch = Filter.Result.DENY;
        return LevelRangeFilter.createFilter(minLevel, maxLevel, match, mismatch);
    }
}
```

#### 定义 Layout

可以看到，上面实现的 Appender 实际上是对官方 HttpAppender 的一个封装，只是在日志输出前后加入自己的逻辑。业务中想给邮件告警的内容增加一些 HTML  的样式，下面需要实现一个自定义的 Layout 来达成这个需求，类似的，给自定义的类加入一些注解：

```java
// 1、继承 AbstractStringLayout 类并重写 toSerializable 方法
// 2、使用 @Plugin 注解，name 的值会配置在 log4j2 的配置文件中，名称可以不与类名一致，这里 category、elementType、printObject 是固定值
@Plugin(name = "HTTPLayout", category = Node.CATEGORY, elementType = Layout.ELEMENT_TYPE, printObject = true)
public class MyHTTPLayout extends AbstractStringLayout {

    
    public HTTPAlertLayout(Charset charset) {
        super(charset);
    }

	// 实现工厂方法 
    @PluginFactory
    public static HTTPAlertLayout createLayout(@PluginAttribute(value = "charset", defaultString = "UTF-8") Charset charset) {
        return new HTTPAlertLayout(charset);
    }

	// 实际上是拼装接口所需要的参数
    @Override
    public String toSerializable(LogEvent event) {
        StackTraceElement source = event.getSource();
        String position = source.getClassName() + source.getMethodName() + ":" + source.getLineNumber();
        String content = "your alert content with html/css style";
        Map<String, String> param = new HashMap<>();
        param.put("content", content);
        param.put("title", "error alert");
        param.put("from", "from@gmail.com");
        param.put("receivers", "to@gmail.com");
        return JSON.toJSONString(param);
    }

	// 因为接口接收的 ContentType 为 json，在此指定
    @Override
    public String getContentType() {
        return "application/json";
    }
}
```

官方对 Layout 也定义了常见格式的实现类，如 CSV、HTML、JSON 等，可以按需应用。

#### 在配置文件中声明 Appender 

定义好 Appender 与 Layout 之后，需要在 log4j2.xml 中配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="info" monitorInterval="15" packages="com.my.package">

    <Appenders>
        <!-- 配置 HTTPAlert -->
        <HTTPAlert name="HTTPAlertAppender" ignoreExceptions="false">
            <HTTPLayout charset="UTF-8"/>
        </HTTPAlert>
        
    </Appenders>

    <Loggers>
        <AsyncRoot level="info" includeLocation="true">
            <AppenderRef ref="HTTPAlertAppender"/>
        </AsyncRoot>
    </Loggers>
</Configuration>
```

如上配置之后，自定义的 Appender 就可以拦截到 error 级别的日志输出。

### 解耦 & 建造者模式

在实现自定义 Appender 的过程中，看到框架实现的每个 Appender 都使用了建造者模式以及功能的解耦设计，这里以官方实现的 SmtpAppender 的源码为例做个介绍。

#### 解耦设计

SmtpAppender 并不是实际进行日志输出的类，而是使用 SmtpManager 进行输出：

```java
@Plugin(name = "SMTP", category = Core.CATEGORY_NAME, elementType = Appender.ELEMENT_TYPE, printObject = true)
public final class SmtpAppender extends AbstractAppender {
    
    // 该 manager 实现了具体的邮件发送逻辑（服务器连接、发送）
    private final SmtpManager manager;
    
    @Override
    public void append(final LogEvent event) {
        manager.sendEvents(getLayout(), event);
    }
}
```

#### 建造者模式

SmtpAppender 的实例化方法使用了 private 修饰，意味着不能使用传统的 new 方法创建实例，而是需要使用内部实现的建造者构建：

```java
@Plugin(name = "SMTP", category = Core.CATEGORY_NAME, elementType = Appender.ELEMENT_TYPE, printObject = true)
public final class SmtpAppender extends AbstractAppender {
  
		// 建造者   
    public static class Builder extends AbstractAppender.Builder<Builder>
            implements org.apache.logging.log4j.core.util.Builder<SmtpAppender> {
        
        @PluginBuilderAttribute
        private String to;

        @PluginBuilderAttribute
        private String from;

				// ...忽略其他属性      

        // 建造者实例化方法，设置对应的参数，如收件人、发件人、SSL 参数等
        @Override
        public SmtpAppender build() {
            if (getLayout() == null) {
                setLayout(HtmlLayout.createDefaultLayout());
            }
            if (getFilter() == null) {
                setFilter(ThresholdFilter.createFilter(null, null, null));
            }
            final SmtpManager smtpManager = SmtpManager.getSmtpManager(getConfiguration(), to, cc, bcc, from, replyTo,
                    subject, smtpProtocol, smtpHost, smtpPort, smtpUsername, smtpPassword, smtpDebug,
                    getFilter().toString(), bufferSize, sslConfiguration);
            return new SmtpAppender(getName(), getFilter(), getLayout(), smtpManager, isIgnoreExceptions(), getPropertyArray());
        }
    }
}
```

这里，设计模式使功能更好维护、粒度拆得更小更好测试，代码的阅读性更佳；业务中若有类似的功能结构，值得借鉴与应用。

### 参考

官方 Appender 介绍：https://logging.apache.org/log4j/2.x/manual/appenders.html#

自定义 Appender & Layout  指引：https://logging.apache.org/log4j/2.x/manual/extending.html