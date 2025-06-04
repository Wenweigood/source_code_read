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

## XML解析

### 背景知识

- `spring.handlers`文件是Spring框架中用于实现SPI（Service Provider Interface）机制的关键配置文件之一，主要用于处理XML命名空间与自定义标签的解析。以下是其核心作用及实现原理的总结：
  - **自定义标签解析**：`spring.handlers`文件定义了XML命名空间（`namespaceUri`）与对应的`NamespaceHandler`实现类的映射关系。当Spring解析XML配置文件时，遇到**非默认命名空间的标签**（如`<dubbo:service ... />`），会通过该文件找到对应的处理器类进行解析。
  - **扩展Spring功能**：第三方框架（如MyBatis、Dubbo）通过此文件与Spring集成，无需修改Spring源码即可注入自定义Bean。



### 解析

```java
// 处理xml的handler入口
public class DubboNamespaceHandler extends NamespaceHandlerSupport implements ConfigurableSourceBeanMetadataElement {
    public DubboNamespaceHandler() {
    }

    public void init() {
        // 初始化各种类型Bean的解析器Parser
        this.registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class));
        this.registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class));
        this.registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class));
        this.registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class));
        this.registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class));
        this.registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class));
        this.registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class));
        this.registerBeanDefinitionParser("tracing", new DubboBeanDefinitionParser(TracingConfig.class));
        this.registerBeanDefinitionParser("ssl", new DubboBeanDefinitionParser(SslConfig.class));
        this.registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class));
        this.registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class));
        this.registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class));
        this.registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class));
        this.registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class));
        this.registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
    
    // 遇到特定标签名时进行解析，如dubbo:service，此时会调用servive对应的解析器
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        BeanDefinitionRegistry registry = parserContext.getRegistry();
        this.registerAnnotationConfigProcessors(registry);
        DubboSpringInitializer.initialize(parserContext.getRegistry());
        BeanDefinition beanDefinition = super.parse(element, parserContext);// 调用对应解析器解析出Bean
        this.setSource(beanDefinition);
        return beanDefinition;
    }

    private void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(registry);
    }
}
```

```java
// 解析逻辑中，根据各个属性如“ref”进行对应解析
private static RootBeanDefinition parse(
            Element element, ParserContext parserContext, Class<?> beanClass, boolean registered) {
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    if (ServiceBean.class.equals(beanClass)) {
        beanDefinition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
    }
    // config id
    String configId = resolveAttribute(element, "id", parserContext);
    if (StringUtils.isNotEmpty(configId)) {
        beanDefinition.getPropertyValues().addPropertyValue("id", configId);
    }

    String configName = "";
    // get configName from name
    if (StringUtils.isEmpty(configId)) {
        configName = resolveAttribute(element, "name", parserContext);
    }

    String beanName = configId;
    if (StringUtils.isEmpty(beanName)) {
        // generate bean name
        String prefix = beanClass.getName();
        int counter = 0;
        beanName = prefix + (StringUtils.isEmpty(configName) ? "#" : ("#" + configName + "#")) + counter;
        while (parserContext.getRegistry().containsBeanDefinition(beanName)) {
            beanName = prefix + (StringUtils.isEmpty(configName) ? "#" : ("#" + configName + "#")) + (counter++);
        }
    }
    beanDefinition.setAttribute(BEAN_NAME, beanName);

    if (ProtocolConfig.class.equals(beanClass)) {
        //            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
        //                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
        //                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
        //                if (property != null) {
        //                    Object value = property.getValue();
        //                    if (value instanceof ProtocolConfig && beanName.equals(((ProtocolConfig)
        // value).getName())) {
        //                        definition.getPropertyValues().addPropertyValue("protocol", new
        // RuntimeBeanReference(beanName));
        //                    }
        //                }
        //            }
    } else if (ServiceBean.class.equals(beanClass)) {
        String className = resolveAttribute(element, "class", parserContext);
        if (StringUtils.isNotEmpty(className)) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            parseProperties(element.getChildNodes(), classDefinition, parserContext);
            beanDefinition
                    .getPropertyValues()
                    .addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, beanName + "Impl"));
        }
    }

    Map<String, Class> beanPropTypeMap = beanPropsCache.get(beanClass.getName());
    if (beanPropTypeMap == null) {
        beanPropTypeMap = new HashMap<>();
        beanPropsCache.put(beanClass.getName(), beanPropTypeMap);
        if (ReferenceBean.class.equals(beanClass)) {
            // extract bean props from ReferenceConfig
            getPropertyMap(ReferenceConfig.class, beanPropTypeMap);
        } else {
            getPropertyMap(beanClass, beanPropTypeMap);
        }
    }

    ManagedMap parameters = null;
    Set<String> processedProps = new HashSet<>();
    for (Map.Entry<String, Class> entry : beanPropTypeMap.entrySet()) {
        String beanProperty = entry.getKey();
        Class type = entry.getValue();
        String property = StringUtils.camelToSplitName(beanProperty, "-");
        processedProps.add(property);
        if ("parameters".equals(property)) {
            parameters = parseParameters(element.getChildNodes(), beanDefinition, parserContext);
        } else if ("methods".equals(property)) {
            parseMethods(beanName, element.getChildNodes(), beanDefinition, parserContext);
        } else if ("arguments".equals(property)) {
            parseArguments(beanName, element.getChildNodes(), beanDefinition, parserContext);
        } else {
            String value = resolveAttribute(element, property, parserContext);
            if (StringUtils.isNotBlank(value)) {
                value = value.trim();
                if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                    RegistryConfig registryConfig = new RegistryConfig();
                    registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                    // see AbstractInterfaceConfig#registries, It will be invoker setRegistries method when
                    // BeanDefinition is registered,
                    beanDefinition.getPropertyValues().addPropertyValue("registries", registryConfig);
                    // If registry is N/A, don't init it until the reference is invoked
                    beanDefinition.setLazyInit(true);
                } else if ("provider".equals(property)
                        || "registry".equals(property)
                        || ("protocol".equals(property)
                                && AbstractServiceConfig.class.isAssignableFrom(beanClass))) {
                    /**
                     * For 'provider' 'protocol' 'registry', keep literal value (should be id/name) and set the value to 'registryIds' 'providerIds' protocolIds'
                     * The following process should make sure each id refers to the corresponding instance, here's how to find the instance for different use cases:
                     * 1. Spring, check existing bean by id, see{@link ServiceBean#afterPropertiesSet()}; then try to use id to find configs defined in remote Config Center
                     * 2. API, directly use id to find configs defined in remote Config Center; if all config instances are defined locally, please use {@link org.apache.dubbo.config.ServiceConfig#setRegistries(List)}
                     */
                    beanDefinition.getPropertyValues().addPropertyValue(beanProperty + "Ids", value);
                } else {
                    Object reference;
                    if (isPrimitive(type)) {
                        value = getCompatibleDefaultValue(property, value);
                        reference = value;
                    } else if (ONRETURN.equals(property) || ONTHROW.equals(property) || ONINVOKE.equals(property)) {
                        int index = value.lastIndexOf(".");
                        String ref = value.substring(0, index);
                        String method = value.substring(index + 1);
                        reference = new RuntimeBeanReference(ref);
                        beanDefinition.getPropertyValues().addPropertyValue(property + METHOD, method);
                    } else if (EXECUTOR.equals(property)) {
                        reference = new RuntimeBeanReference(value);
                    } else {
                        if ("ref".equals(property)
                                && parserContext.getRegistry().containsBeanDefinition(value)) {
                            BeanDefinition refBean =
                                    parserContext.getRegistry().getBeanDefinition(value);
                            if (!refBean.isSingleton()) {
                                throw new IllegalStateException(
                                        "The exported service ref " + value + " must be singleton! Please set the "
                                                + value + " bean scope to singleton, eg: <bean id=\"" + value
                                                + "\" scope=\"singleton\" ...>");
                            }
                        }
                        reference = new RuntimeBeanReference(value);
                    }
                    if (reference != null) {
                        beanDefinition.getPropertyValues().addPropertyValue(beanProperty, reference);
                    }
                }
            }
        }
    }

    NamedNodeMap attributes = element.getAttributes();
    int len = attributes.getLength();
    for (int i = 0; i < len; i++) {
        Node node = attributes.item(i);
        String name = node.getLocalName();
        if (!processedProps.contains(name)) {
            if (parameters == null) {
                parameters = new ManagedMap();
            }
            String value = node.getNodeValue();
            parameters.put(name, new TypedStringValue(value, String.class));
        }
    }
    if (parameters != null) {
        beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }

    // post-process after parse attributes
    if (ProviderConfig.class.equals(beanClass)) {
        parseNested(
                element, parserContext, ServiceBean.class, true, "service", "provider", beanName, beanDefinition);
    } else if (ConsumerConfig.class.equals(beanClass)) {
        parseNested(
                element,
                parserContext,
                ReferenceBean.class,
                true,
                "reference",
                "consumer",
                beanName,
                beanDefinition);
    } else if (ReferenceBean.class.equals(beanClass)) {
        configReferenceBean(element, parserContext, beanDefinition, null);
    } else if (MetricsConfig.class.equals(beanClass)) {
        parseMetrics(element, parserContext, beanDefinition);
    }

    // register bean definition
    if (parserContext.getRegistry().containsBeanDefinition(beanName)) {
        throw new IllegalStateException("Duplicate spring bean name: " + beanName);
    }

    if (registered) {
        parserContext.getRegistry().registerBeanDefinition(beanName, beanDefinition);
    }
    return beanDefinition;
}
```

























