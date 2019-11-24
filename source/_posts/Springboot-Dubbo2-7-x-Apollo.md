---
title: Springboot + Dubbo2.7.x + Apollo
date: 2019-11-24 15:02:37
tags: 配置中心
---
# 环境准备
1. Java 1.8
2. Apollo配置中心安装，[Github上有详细的安装过程](https://github.com/ctripcorp/apollo/wiki/Quick-Start)
3. Zookeeper注册中心，[官网下载和安装](https://zookeeper.apache.org/releases.html)

# 开始搭建

## 1. 创建配置

### 1.1 创建项目
前面安装好Apollo后，从浏览器进入配置中心管理页面(默认端口8070)
![](/images/ConfigCenter/20191124151846.png)
如上图创建3个Project，分别为demo-common（公共配置）、demo-provider（提供者配置）、demo-consumer（消费者配置）
### 1.2 创建公共Namespace
先进入demo-common项目，点击左下角的添加 Add Namespace 按钮
![](/images/ConfigCenter/20191124152637.png)
进入添加Namespace页面后点击 Create Namespace 按钮
![](/images/ConfigCenter/20191124152954.png)
创建一个dubbo的Namespace，然后回到demo-common的项目配置中，此时项目多了一个dubbo的公共配置。然后在dubbo的Namespace下面添加如下配置
```java
dubbo.protocol.name = dubbo
dubbo.registry.address = zookeeper://zookeeper-ip:2181
dubbo.registry.simplified = true
```
然后点击 dubbo 上的 Realease 按钮

### 1.3 添加私有配置
#### 1.3.1 添加服务端的配置
- 添加上面创建的dubbo的Namespace，进入 demo-provider项目，点击 Add Namespace按钮，然后在下拉选项中选择 dubbo，然后 Submit
![](/images/ConfigCenter/20191124154143.png)
- 回到 demo-provider 项目中，在 application 的 Namespace中添加以下配置，然后 Release
```Java
dubbo.application.name = demo-provider
dubbo.protocol.port = 20880
dubbo.scan.base-packages = com.dzeb.demo.service # dubbo的Service注解的类的包
```

#### 1.3.2 添加消费端的配置
- 跟上面 demo-provider 一样，进入 demo-consumer, 先添加公共的 dubbo 的 Namespace
- 在 application 的 Namespace 中添加以下配置，然后 Release
```Java
dubbo.application.name = demo-consumer
```

## 2. 创建Java工程
### 2.1 创建Java工程
创建 demo-provider、demo-consumer、demo-api 三个maven工程
### 2.2 添加Maven依赖
demo-provider的pom.xml：
```Java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <version>2.2.1.RELEASE</version>
    </dependency>
    <!-- apollo 依赖 -->
    <dependency>
        <groupId>com.ctrip.framework.apollo</groupId>
        <artifactId>apollo-client</artifactId>
        <version>1.5.0</version>
    </dependency>
    <!-- zookeeper 需要用的依赖 -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>4.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>4.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.5.5</version>
    </dependency>
    <!-- dubbo 和 springboot 集成依赖 -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
        <version>2.7.4.1</version>
    </dependency>
    <!-- demo-api依赖 -->
    <dependency>
        <groupId>com.dzeb</groupId>
        <artifactId>demo-api</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```

demo-consumer 的 pom.xml：
```java
<dependencies>
    <!-- springboot web启动 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.2.1.RELEASE</version>
    </dependency>
    <!-- Apollo -->
    <dependency>
        <groupId>com.ctrip.framework.apollo</groupId>
        <artifactId>apollo-client</artifactId>
        <version>1.5.0</version>
    </dependency>
    <!-- dubbo -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
        <version>2.7.4.1</version>
    </dependency>
    <!-- zookeeper 需要用到的依赖-->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>4.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>4.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.5.5</version>
    </dependency>
    <!-- demo-api -->
    <dependency>
        <groupId>com.dzeb</groupId>
        <artifactId>demo-api</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```
### 2.3 配置使用 Apollo 
demo-provider 的 resources/application.properties：
```Java
app.id = demo-provider # Apollo配置中心中的 demo-provider 的 appId
apollo.meta=http://apollo-ip:8080 # 配置中心的meta-server，默认8080端口
apollo.bootstrap.enabled = true # 注入默认application namespace的配置示例
apollo.bootstrap.namespaces = application,dubbo # 要使用的 Namespace
```
demo-consumer 的 resources/application.properties：
```java
app.id = demo-consumer
apollo.meta=http://apollo-ip:8080
apollo.bootstrap.enabled = true
apollo.bootstrap.namespaces = application,dubbo
```

### 2.4 创建测试类
demo-api 工程创建类 DemoService:
```java
public interface DemoService {
    String sayHello(String name);
}
```
demo-provider 工程创建类 DefaultDemoService 和 DemoProviderApplication：
```Java
import org.apache.dubbo.config.annotation.Service;
import org.springframework.beans.factory.annotation.Value;

// 这里的 Service 是 org.apache.dubbo.config.annotation 包下面的，不是 Spring 的 Service 注解
@Service(version = "1.0.0")
public class DefaultDemoService implements DemoService {

    /**
     * application name
     */
    @Value("${dubbo.application.name}")
    private String serviceName;

    public String sayHello(String name) {
        return String.format("[%s] : Hello, %s", serviceName, name);
    }
}
```
```Java
@SpringBootApplication
public class DemoProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoProviderApplication.class, args);
    }
}
```
demo-consumer 工程创建类 DemoController 和 DemoApiApplication：
```Java
@RestController
@RequestMapping
public class DemoController {

    @Reference(version = "1.0.0")
    private DemoService demoService;

    @RequestMapping("sayHello/{name}")
    public ResponseEntity<String> sayHello(@PathVariable("name") String name) {
        return ResponseEntity.ok().body(demoService.sayHello(name));
    }
}
```
```java
@SpringBootApplication
public class DemoApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApiApplication.class, args);
    }
}
```

### 2.5 启动访问
分别启动 DemoProviderApplication 和 DemoApiApplication，然后在浏览器或HTTP测试工具上访问，如下
![](/images/ConfigCenter/20191124162922.png)

# 填坑
如果启动遇到如下报错：
```Java
java.lang.IllegalStateException: zookeeper not connected
```
有一种情况是超时导致的，可以在zookeeper配置的连接增加timeout（default 5000），然后 Release
```Java
zookeeper://zookeeper-ip:2181?timeout=10000
```
# 总结
这次搭建使用了最新的dubbo 2.7.x，其中有许多改动，比如默认配置中心和移除 zkclient 的实现：
```java
为了兼容2.6.x版本配置，在使用Zookeeper作为注册中心，且没有显示配置配置中心的情况下，Dubbo框架会默认将此Zookeeper用作配置中心，但将只作服务治理用途。
```
```Java
注意:在2.7.x的版本中已经移除了zkclient的实现,如果要使用zkclient客户端,需要自行拓展
```

# 源码地址
https://github.com/dingzebin/dubbo-apollo-demo

# 参考
[Dubbo官方文档](https://dubbo.apache.org/zh-cn/docs)

[Apollo官方文档](https://github.com/ctripcorp/apollo)
