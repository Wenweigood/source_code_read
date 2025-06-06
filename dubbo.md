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
  	<!-- 通过class指定实现类亦可
  	<dubbo:service interface="org.example.dubbodemo.api.HelloWorldService" class="org.example.exampleproject.service.HelloWorldServiceImpl"/>
		-->
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
// 遇到对应标签时，会自动调用对应解析器解析，处理内含属性值，添加到BeanDefinition的属性列表中，后续实例化时自动依赖注入属性
// com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser#parse
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    String id = element.getAttribute("id");
  	// 计算Bean id
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                generatedBeanName = element.getAttribute("interface");
            }
        }
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        int counter = 2;
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            id = generatedBeanName + (counter++);
        }
    }
  	// 检查是否有重复Bean id
    if (id != null && id.length() > 0) {
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    if (ProtocolConfig.class.equals(beanClass)) {
        for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
            BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
            PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
            if (property != null) {
                Object value = property.getValue();
                if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                    definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                }
            }
        }
    } else if (ServiceBean.class.equals(beanClass)) {
      	// 如果指定了class，则实现类不必加@Service注解，而是会在此处自动实例化并注册到Spring容器
      	// 如果是ref方式指定，则是引用已存在的Spring Bean
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            parseProperties(element.getChildNodes(), classDefinition);
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
    } else if (ProviderConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
    } else if (ConsumerConfig.class.equals(beanClass)) {
        parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
    }
    Set<String> props = new HashSet<String>();
    ManagedMap parameters = null;
  	// 从set方法找起，找对应的字段
    for (Method setter : beanClass.getMethods()) {
        String name = setter.getName();
        if (name.length() > 3 && name.startsWith("set")
                && Modifier.isPublic(setter.getModifiers())
                && setter.getParameterTypes().length == 1) {
            Class<?> type = setter.getParameterTypes()[0];
          	// 字段名 = 去掉set + 将首字母小写
            String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
            props.add(property);
            Method getter = null;
            try {
                getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
            } catch (NoSuchMethodException e) {
                try {
                    getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e2) {
                }
            }
            if (getter == null
                    || !Modifier.isPublic(getter.getModifiers())
                    || !type.equals(getter.getReturnType())) {
                continue;
            }
            if ("parameters".equals(property)) {
                parameters = parseParameters(element.getChildNodes(), beanDefinition);
            } else if ("methods".equals(property)) {
                parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
            } else if ("arguments".equals(property)) {
                parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
            } else {
                String value = element.getAttribute(property);
                if (value != null) {
                    value = value.trim();
                    if (value.length() > 0) {
                        if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                            RegistryConfig registryConfig = new RegistryConfig();
                            registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                        } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("registries", value, beanDefinition, parserContext);
                        } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("providers", value, beanDefinition, parserContext);
                        } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                            parseMultiRef("protocols", value, beanDefinition, parserContext);
                        } else {
                            ...
                          	// 向 BeanDefinition 的属性列表（PropertyValues）中添加一个属性值，用于后续 Bean 实例化时的依赖注入，比如 <dubbo:registry address="zookeeper://127.0.0.1:2181"/> 中的address，后续会被注入RegistryConfig的实例字段中
                            beanDefinition.getPropertyValues().addPropertyValue(property, reference);// 
                        }
                    }
                }
            }
        }
    }
    ...
    return beanDefinition;
}
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
// ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware
...

@Override
@SuppressWarnings({"unchecked", "deprecation"})
public void afterPropertiesSet() throws Exception {
  	// 给ProviderConfig赋值
  	// <dubbo:provider> 是 Dubbo 框架中用于配置服务提供者缺省值的 XML 标签，主要作用是为 <dubbo:service> 和 <dubbo:protocol> 标签提供全局默认配置，避免重复定义相同属性
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
  	// <dubbo:application> 是 Dubbo 框架中用于配置当前应用信息的核心 XML 标签，主要作用是为服务提供者（Provider）和消费者（Consumer）定义全局标识和基础属性
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
    ...
  	// 从RegistryConfig的实例中获取注册中心配置（RegistryConfig的实例字段如address则已在XML解析时，放到了BeanDefinition的属性字段中，在RegistryConfig实例化时自动注入）
    // 根据if判断条件可知，注册中心的配置优先级：显式配置（当前对象） > Provider配置 > Application配置 > 容器中动态注册的默认配置
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
    ...
    // <dubbo:protocol> 是 Dubbo 框架中用于配置服务提供者通信协议的核心 XML 标签，其作用是为服务暴露（export）指定传输协议（如 dubbo、http、hessian）、端口及性能调优参数（如线程池类型、线程池大小）
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
  	// 是否延迟暴露，若延迟暴露，则在Spring容器完成所有Bean初始化后，通过发布ContextRefreshedEvent事件来回调com.alibaba.dubbo.config.spring.ServiceBean#onApplicationEvent暴露服务
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

## 暴露服务

```java
// 暴露服务核心方法com.alibaba.dubbo.config.ServiceConfig#doExport，加锁执行
// 由此可以看出，Dubbo服务是以单个服务（Service）为维度暴露服务的。每个服务的暴露独立管理，符合微服务架构的细粒度治理需求
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
    }
    checkDefault();
    if (provider != null) {
        if (application == null) {
            application = provider.getApplication();
        }
        if (module == null) {
            module = provider.getModule();
        }
        if (registries == null) {
            registries = provider.getRegistries();
        }
        if (monitor == null) {
            monitor = provider.getMonitor();
        }
        if (protocols == null) {
            protocols = provider.getProtocols();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
    if (ref instanceof GenericService) {
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            generic = Boolean.TRUE.toString();
        }
    } else {
        try {
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
        checkRef();
        generic = Boolean.FALSE.toString();
    }
  	// 本地存根（Local Stub）是一种在消费端（Consumer）执行的代理逻辑，由服务提供方（Provider）定义并嵌入到客户端，用于在远程调用前后执行额外操作
  	// 主要功能：参数校验或预处理、容错与降级、本地缓存、AOP式增强
    if (local != null) {
        if ("true".equals(local)) {
            local = interfaceName + "Local";
        }
        Class<?> localClass;
        try {
            localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(localClass)) {
            throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
        }
    }
    if (stub != null) {
        if ("true".equals(stub)) {
            stub = interfaceName + "Stub";
        }
        Class<?> stubClass;
        try {
            stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        if (!interfaceClass.isAssignableFrom(stubClass)) {
            throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
        }
    }
    checkApplication();
    checkRegistry();
    checkProtocol();
    appendProperties(this);
    checkStubAndMock(interfaceClass);
  	// 路径默认为接口名
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }
  	// 对外暴露
    doExportUrls();
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```

```java
// 如果指定了多个<dubbo:protocol>，则会以多种协议，在各个ZK上都暴露服务
private void doExportUrls() {
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
  	// 默认使用dubbo协议
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
  	// 标明是服务方
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
  	// 装载基本元数据
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);
  	// 装载方法名、参数
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            List<ArgumentConfig> arguments = method.getArguments();
            if (arguments != null && !arguments.isEmpty()) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    if (argument.getIndex() != -1) {
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        }
    }

    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }
    // export service
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }
		
		// 获取服务绑定的主机地址
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
  	// 获取服务绑定的端口号
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
  	// 构建暴露的URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    // don't export when none is configured
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
          	// 本地暴露（服务提供方、调用方位于同一JVM进程）使用injvm协议，避免不必要的网络开销，省去网络传输、序列化/反序列化等步骤，性能显著优于远程调用
            exportLocal(url);
        }
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                    }
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
										// 根据具体协议头如dubbo://调用指定protocol实例
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
								// 根据具体协议头如dubbo://调用指定protocol实例
                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```

### 指定端口开放

```java
// dubbo协议的export，com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#export
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
		// 指定端口上启动一个网络服务（默认Netty），用于接收消费者发起的请求
    openServer(url);
    optimizeSerialization(url);
    return exporter;
}
```

### 注册服务到ZK

```java
// RegistryProtocol的export，真正注册服务到ZK，com.alibaba.dubbo.registry.integration.RegistryProtocol#export
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    URL registryUrl = getRegistryUrl(originInvoker);

    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    boolean register = registedProviderUrl.getParameter("register", true);

    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

    if (register) {
      	// 向zk注册服务，registryUrl例子：
      	// zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.6.2&export=dubbo%3A%2F%2F172.20.10.3%3A20880%2Forg.example.exampleproject.api.HelloWorldService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D172.20.10.3%26bind.port%3D20880%26dubbo%3D2.6.2%26generic%3Dfalse%26interface%3Dorg.example.exampleproject.api.HelloWorldService%26methods%3DsayHello%26pid%3D9171%26side%3Dprovider%26timestamp%3D1749196316396&pid=9171&timestamp=1749196316394
        register(registryUrl, registedProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
}
```

















