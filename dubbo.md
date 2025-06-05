# Dubbo

```xml
<!-- 对应dubbo 2.6.2，经典版本便于分析 -->
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>
```



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

## XML解析

### 背景知识

- `spring.handlers`文件是Spring框架中用于实现SPI（Service Provider Interface）机制的关键配置文件之一，主要用于处理XML命名空间与自定义标签的解析。以下是其核心作用及实现原理的总结：
  - **自定义标签解析**：`spring.handlers`文件定义了XML命名空间（`namespaceUri`）与对应的`NamespaceHandler`实现类的映射关系。当Spring解析XML配置文件时，遇到**非默认命名空间的标签**（如`<dubbo:service ... />`），会通过该文件找到对应的处理器类进行解析。
  - **扩展Spring功能**：第三方框架（如MyBatis、Dubbo）通过此文件与Spring集成，无需修改Spring源码即可注入自定义Bean。

### 文件解析

```java
// 处理xml的handler入口，继承自NamespaceHandlerSupport，遇到特定标签名时进行解析，如dubbo:service，自动调用已经注册的Parser，执行parse()方法
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
  ...
  // 初始化各种类型Bean的解析器Parser
  public void init() {
      this.registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
      this.registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
      this.registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
      this.registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
      this.registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
      this.registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
      this.registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
      this.registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));// ServiceBean是提供服务的Bean，实现了Spring的InitializingBean接口，会在Spring容器初始化时调用afterPropertiesSet()
      this.registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));// ReferenceBean是引用服务的Bean，实现了Spring的FactoryBean接口，通过 FactoryBean 机制动态生成代理对象，实现透明化的远程调用
      this.registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
  }
  ...
}
```

```java

```

## Bean初始化

### 背景知识

- `InitializingBean` 是 Spring 框架提供的一个生命周期接口，用于在 Bean 的属性注入完成后执行自定义初始化逻辑。其核心作用包括
  - **初始化回调**：定义 `afterPropertiesSet()` 方法，Spring 会在 Bean 的属性注入完成后自动调用该方法
  - **执行顺序**：在 Spring 生命周期中，`afterPropertiesSet()` 的调用时机位于属性注入之后、`@PostConstruct` （Java标准注解）注解方法之后、自定义 `init-method` 之前
- `DisposableBean`
  - **作用**：在Bean销毁前执行清理逻辑
  - **触发时机**：Spring容器关闭时，在`destroy-method`之前执行
- `ApplicationListener<ContextRefreshedEvent>`
  - **作用**：监听Spring上下文刷新完成事件
  - **设计意义**：确保服务在Spring完全初始化后才暴露，避免：
    - 依赖未就绪
    - 配置未完全加载
    - AOP代理未生成

```java
// 提供服务核心Bean，实现了多个接口，在Spring Bean的生命周期各个节点起作用
ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware
...

@Override
@SuppressWarnings({"unchecked", "deprecation"})
public void afterPropertiesSet() throws Exception {
  	// 给ProviderConfig赋值
    if (getProvider() == null) {
        Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
        if (providerConfigMap != null && providerConfigMap.size() > 0) {
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                    && providerConfigMap.size() > 1) { // backward compatibility
                List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() != null && config.isDefault().booleanValue()) {
                        providerConfigs.add(config);
                    }
                }
                if (!providerConfigs.isEmpty()) {
                    setProviders(providerConfigs);
                }
            } else {
                ProviderConfig providerConfig = null;
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (providerConfig != null) {
                            throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                        }
                        providerConfig = config;
                    }
                }
                if (providerConfig != null) {
                    setProvider(providerConfig);
                }
            }
        }
    }
  	// 
    if (getApplication() == null
            && (getProvider() == null || getProvider().getApplication() == null)) {
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    if (getModule() == null
            && (getProvider() == null || getProvider().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
  	// 
    if ((getRegistries() == null || getRegistries().isEmpty())
            && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
            && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    registryConfigs.add(config);
                }
            }
            if (registryConfigs != null && !registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    if (getMonitor() == null
            && (getProvider() == null || getProvider().getMonitor() == null)
            && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    if ((getProtocols() == null || getProtocols().isEmpty())
            && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
        Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
        if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
            List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
            for (ProtocolConfig config : protocolConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    protocolConfigs.add(config);
                }
            }
            if (protocolConfigs != null && !protocolConfigs.isEmpty()) {
                super.setProtocols(protocolConfigs);
            }
        }
    }
  	// 默认路径为接口全限定名
    if (getPath() == null || getPath().length() == 0) {
        if (beanName != null && beanName.length() > 0
                && getInterface() != null && getInterface().length() > 0
                && beanName.startsWith(getInterface())) {
            setPath(beanName);
        }
    }
    if (!isDelay()) {
      	// 暴露服务
        export();
    }
}

@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}

...
```

## 对外暴露服务

















