# 接收SMS发送报告-使用Java和Spark

原始教程地址：
https://www.nexmo.com/blog/2019/07/26/receiving-sms-delivery-receipts-with-java-and-spark-dr


## 发送报告

使用SMS API发送短信后，得到`status:0`的结果即表示短信已经发送给运营商。但除非可以与接收人确认，否则如何能得知短信送达情况？Nexmo所使用的大部分SMS发送网络都支持发送报告，用于反馈短信接收情况，这里介绍如何接收它们。如果各位还不知道如何使用SMS API发送短信，请参考如下文档：
https://www.nexmo.com/blog/2017/05/03/send-sms-messages-with-java-dr

首先必须降低各位的期待，不是所有的发送报告都是准确的，在某些国家和地区会更精确，某些则不然。一些运营商会反馈他们是否接收，某些会反馈不准确的信息，某些则根本不反馈任何信息。因此在相信这些发送报告前，先看看以下文档：
https://help.nexmo.com/hc/en-us/articles/204014863-What-will-I-receive-if-a-network-country-does-not-support-Delivery-Receipts-

为获取短信发送报告，需要创建一个公开的WebHook，并配置Nexmo账户或号码去使用它。这里使用Java和Spark架构来创建这个WebHook，源代码在GitHub的如下文档库中有所分享：
https://github.com/Nexmo/java-get-delivery-receipt

## 前提条件

* Nexmo账户，需要准备API的key，即账户，和secret，即密码，以及虚拟号码：
https://dashboard.nexmo.com/sign-up
* Java开发包，版本在8以上，这里使用11，用于支持Java开发环境
https://www.oracle.com/technetwork/java/javase/downloads/index.html
-> 或者 https://openjdk.java.net/install/
* Gradle，版本在3.4以上，用于创建项目和管理依赖关系
https://gradle.org/install/
* ngrok，用于将本地的webhook发布到公共网络环境，即提供一个公有URL
https://ngrok.com/
* IDE，即一个即成开发环境，我使用的是MS Visual Studio Code，各位自便

### 创建项目

以下语句建立项目文件夹，进入项目文件夹，并初始化项目：
```
$ mkdir get-delivery-receipt
$ cd get-delivery-receipt
$ gradle init --type=java-application
Starting a Gradle Daemon (subsequent builds will be faster)
BUILD SUCCESSFUL in 3s
2 actionable tasks: 2 executed
```

### 依赖关系

更新`build.gradle`文件，更新`jcenter()`为`mavenCentral()`，更新`dependencies`部分为如下结果：
```
dependencies {
    // Spark framework
    implementation 'com.sparkjava:spark-core:2.8.0'
 
    // Nexmo client library
    implementation 'com.nexmo:client:4.4.0'
 
    // To display formatted JSON
    implementation 'com.cedarsoftware:json-io:4.10.1'
}

repositories {
    // Use jcenter for resolving your dependencies.
    // You can declare any Maven/Ivy/file repository here.
    // jcenter()
    mavenCentral()
}
```

### Web程序

编辑`/get-delivery-receipt/src/main/java`路径下的`App.java`文件：
```
// 添加以下两行代码，导入必须的代码库
import static spark.Spark.*;
import com.cedarsoftware.util.io.JsonWriter;

public class App {
    // 删除以下方法
    // public String getGreeting() {
    //     return "Hello world.";
    // }

    public static void main(String[] args) throws Exception {
        // 删除以下一行代码
        // System.out.println(new App().getGreeting());

        // 使用spark的port方法监听3000端口
        port(3000);
    }
}
```

### GET方法

当Nexmo从运营商获得短信报告后，会检查各位的账户设置，如果得到WebHook的URL定义地址，Nexmo就会向该地址发送一个默认为GET方法的请求。因此在前文编辑过的代码基础上，添加GET方法：
```
// 使用spark的port方法监听3000端口
port(3000);

get("/webhooks/delivery-receipt", (req, res) -> {
    System.out.println("DLR received via GET");
    for (String param : req.queryParams()) {
        System.out.printf("%s: %s\n", param, req.queryParams(param));
    }
    res.status(204);
    return "";
});
```
以上代码解析GET请求，从请求信息中剥离出发送报告信息并输出到标准输出`stdout`。返回的HTTP代码204告诉Nexmo API请求已经成功处理，且不会返回任何内容。

### POST方法

通过配置Nexmo账户，也可以指定使用POST方法提供发送报告，编码在URL中或以JSON的方式，为此继续追加如下代码：
```
post("/webhooks/delivery-receipt", (req, res) -> {
    if (req.contentType().startsWith("application/x-www-form-urlencoded")) {
        System.out.println("DLR received via POST");
        for (String param : req.queryParams()) {
            System.out.printf("%s: %s\n", param, req.queryParams(param));
        }
    } else {
        System.out.println("DLR received via POST-JSON");
        String prettyJson = JsonWriter.formatJson(req.body());
        System.out.println(prettyJson);
    }
    res.status(204);
    return "";
});
```

### 虚拟号码

需要一个虚拟号码用来发送短信，可以从以下用户控制台购买，或使用CLI命令行工具：
https://dashboard.nexmo.com/
准备API的Key和Secret，即用户名和密码。安装CLI工具，设置账户，检索英国SMS功能的虚拟号码，购买一个并确认：
```
$ npm install -g nexmo-cli
$ nexmo setup YOUR_API_KEY YOUR_API_SECRET
$ nexmo number:search GB --sms --verbose
$ nexmo number:buy NEXMO_NUMBER
$ nexmo number:list
```

### 发布WebHook

本地运行的WebHook服务器需要发布到公网才能被Nexmo的API连接到，这里用`ngrok`工具，具体可以到网上搜索使用方法。这里已经完成了安装，直接运行如下命令可以发布本地3000端口到公网：
```
$ ./ngrok http 3000
...
Forwarding                    http://48a0b726.ngrok.io -> localhost:3000        
Forwarding                    https://48a0b726.ngrok.io -> localhost:3000 
...
```
省略无关内容，各位只需要关注以上重定向的两行，其中前面的URL即是开放到公网的本地WebHook服务器地址。由于之前在`GET`和`POST`方法定义中有涉及路径，这里结合`ngrok`的公开地址，完善路径如下：
```
https://48a0b726.ngrok.io/webhooks/delivery-receipt
```
拷贝如上地址，导航到如下控制台页面，更新`Settings -> Default SMS Setting -> Delivery receipts`
https://dashboard.nexmo.com/settings

### 测试

执行以下命令运行程序：
```
$ gradle run
```
在我的测试电脑上有如下错误，程序停止在75%的地方不再运行：
```
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
<=========----> 75% EXECUTING
```
解决办法就是在`build.gradle`文件中添加如下依赖项目：
```
dependencies {
    compile 'org.slf4j:slf4j-log4j12:1.7.21'
}
```
再次执行`gradle run`又会出现如下错误：
```
log4j:WARN No appenders could be found for logger (spark.route.Routes).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
```
解决办法是创建文件`src/main/resources/log4j.properties`.
如果需要输出日志添加如下代码：
```
log4j.rootLogger=DEBUG, console
log4j.logger.xxx=DEBUG, console

log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d [%-5p-%c] %m%n
```
如果不需要输入日志则添加如下代码：
```
log4j.rootLogger=FATAL, null
log4j.appender.null=org.apache.log4j.varia.NullAppender
```
排除以上错误后，程序执行仍然停止在75%的地方就不再前进。

尝试发送短信：
```
$ nexmo sms -f NEXMOTEST YOUR_OWN_NUMBER "This is a test message"
This operation will charge your account.
Please type "confirm" to continue: confirm
Message sent to:   YOUR_OWN_NUMBER
Remaining balance: 9.52250005 EUR
Message price:     0.02930000 EUR
```

### 参考：
WebHook https://developer.nexmo.com/concepts/guides/webhooks
Spark https://sparkjava.com/
ngrok https://www.nexmo.com/blog/2017/07/04/local-development-nexmo-ngrok-tunnel-dr

### 问题
* https://github.com/Nexmo/java-get-delivery-receipt 不可用
* `gradle run`只停止在`75%`的进度，不再前进，程序也无法返回发送日志
* 还有其他错误，之前已经通过修改依赖库定义文件和日志属性定义文件修正