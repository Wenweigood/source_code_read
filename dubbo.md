# Dubbo

## 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://dubbo.apache.org/schema/dubbo
                           http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!-- 应用信息 -->
    <dubbo:application name="demo-provider"/>

    <!-- 注册中心 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>

    <!-- 协议配置 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- 暴露服务 -->
    <dubbo:service interface="org.example.dubbodemo.api.HelloWorldService" ref="helloWorldService"/>
</beans>
```

## 启动

```java
@SpringBootApplication
@ImportResource("classpath:dubbo/dubbo.xml")// 扫描指定路径下的xml文件
public class DubboDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DubboDemoApplication.class, args);
    }
}
```

