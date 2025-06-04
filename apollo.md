# Apollo

## 初始化

### 配置文件

```properties
# Apollo服务地址
apollo.meta=http://localhost:8080
# 应用ID（需与Apollo控制台配置一致）
app.id=your_app_id
# 集群
apollo.cluster=default
# springboot 的 bootstrap阶段注入配置到Environment
apollo.bootstrap.enabled=true
```

### 启动

#### 方式一：Bean加载前

Apollo 的配置会在 Spring Boot 的 `prepareContext` 阶段（即 **Bean 加载前**）通过 `ApolloApplicationContextInitializer` 初始化。配置会被添加到 `Environment` 的 **最优先位置**（`PropertySources` 的头部），确保后续 `@Value`注入或条件注解（如 `@Conditional`）优先使用 Apollo 的配置

- 需要 启动时即加载配置（如数据库连接参数）
- 需要 实时热更新（如动态调整日志级别）

```java
// spring.factories文件指明注入ApolloApplicationContextInitializer.class并执行initialize()
org.springframework.context.ApplicationContextInitializer=\
com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer
```

```java
// namespace为维度获取配置
protected void initialize(ConfigurableEnvironment environment) {
    if (!environment.getPropertySources().contains("ApolloBootstrapPropertySources")) {
        String namespaces = environment.getProperty("apollo.bootstrap.namespaces", "application");
        logger.debug("Apollo bootstrap namespaces: {}", namespaces);
        List<String> namespaceList = NAMESPACE_SPLITTER.splitToList(namespaces);
        CompositePropertySource composite = new CompositePropertySource("ApolloBootstrapPropertySources");
        Iterator i$ = namespaceList.iterator();

        while(i$.hasNext()) {
            String namespace = (String)i$.next();
            Config config = ConfigService.getConfig(namespace);// 读取远端配置入口
            composite.addPropertySource(
              this.configPropertySourceFactory.getConfigPropertySource(namespace, config)
            );
        }
        environment.getPropertySources().addFirst(composite);// 加在头部
    }
}
```

```java
// 获取远端配置的核心源码
public RemoteConfigRepository(String namespace) {
    this.m_namespace = namespace;
    this.m_configCache = new AtomicReference();
    this.m_configUtil = (ConfigUtil)ApolloInjector.getInstance(ConfigUtil.class);
    this.m_httpUtil = (HttpUtil)ApolloInjector.getInstance(HttpUtil.class);
    ... // 一些其他配置
    this.trySync();// 通过HTTP GET请求从Apollo服务端获取配置
    this.schedulePeriodicRefresh();// 通过一个核心线程数为1，最大线程数为Integer.MAX_VALUE的ScheduledThreadPool线程池，以固定速率执行trySync刷新本地缓存（默认5分钟一次）。作为长连接的兜底机制
    this.scheduleLongPollingRefresh();// 长连接服务端，服务端会挂起直到超时或有配置变更。通过单线程线程池SingleThreadPool执行
}
```

#### 方式二：**Bean 加载后**（延迟加载）

Apollo 的配置会在 Spring Boot 的 `refreshContext` 阶段（即 **Bean 加载后**）通过 `PropertySourcesProcessor` 加载。若其他 Bean 在初始化时依赖配置（如 `@Value` 注入），可能因 Apollo 配置未加载而使用本地文件（如 `application.yml`）的默认值

- 配置非启动必需，允许延迟加载
- 与其他配置中心（如 Spring Cloud Config）共存时，需控制加载顺序

```java
// ==必须显式注解==启用Apollo配置，可选加载的namespace
@EnableApolloConfig(value = {"application", "namespace2", "namespace3"})

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({ApolloConfigRegistrar.class}) // 导入类到Spring 容器
public @interface EnableApolloConfig {
    String[] value() default {"application"};

    int order() default Integer.MAX_VALUE;
}
```

```java
public class ApolloConfigRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {
    private final ApolloConfigRegistrarHelper helper = (ApolloConfigRegistrarHelper)ServiceBootstrap.loadPrimary(ApolloConfigRegistrarHelper.class);

    public ApolloConfigRegistrar() {
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	      // 注册一些工具Bean、配置
        this.helper.registerBeanDefinitions(importingClassMetadata, registry);
    }

    public void setEnvironment(Environment environment) {
        this.helper.setEnvironment(environment);
    }
}
```

```java
// 注册工具Bean、配置的源码
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));// 获取注解指定的namespace
    String[] namespaces = attributes.getStringArray("value");
    int order = (Integer)attributes.getNumber("order"); // 默认Integer.MAX_VALUE，最高优先级
    String[] resolvedNamespaces = this.resolveNamespaces(namespaces);// 对指定的namespace进行标签替换（如有）
    PropertySourcesProcessor.addNamespaces(Lists.newArrayList(resolvedNamespaces), order);
    Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap();
    propertySourcesPlaceholderPropertyValues.put("order", 0);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(), PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesProcessor.class.getName(), PropertySourcesProcessor.class);// 将Apollo配置加载到Environment的关键入口
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(), ApolloAnnotationProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(), SpringValueProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(), SpringValueDefinitionProcessor.class);
}
```

```java
public class PropertySourcesProcessor implements BeanFactoryPostProcessor, EnvironmentAware, PriorityOrdered {
    ...
    // 实现了org.springframework.beans.factory.config.BeanFactoryPostProcessor接口，Bean初始化时调用
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        this.initializePropertySources();// 从Apollo服务端读取配置
        this.initializeAutoUpdatePropertiesFeature(beanFactory);// 启用配置的自动更新。通过注册监听器（trySync内回调），在配置变化时通过反射的方式更新值
    }
  	...
}
```

```java
private void initializePropertySources() {
  if (!this.environment.getPropertySources().contains("ApolloPropertySources")) {
      CompositePropertySource composite = new CompositePropertySource("ApolloPropertySources");
      ImmutableSortedSet<Integer> orders = ImmutableSortedSet.copyOf(NAMESPACE_NAMES.keySet());
      Iterator<Integer> iterator = orders.iterator();

      while(iterator.hasNext()) {
          int order = (Integer)iterator.next();
          Iterator i$ = NAMESPACE_NAMES.get(order).iterator();

          while(i$.hasNext()) {
              String namespace = (String)i$.next();
              Config config = ConfigService.getConfig(namespace);// 读取远端配置入口，同上。1、通过HTTP GET请求从Apollo服务端获取配置；2、定时刷新本地缓存；3、长连接服务端获取变更通知
              composite.addPropertySource(
                this.configPropertySourceFactory.getConfigPropertySource(namespace, config)
              );
          }
      }
      NAMESPACE_NAMES.clear();
      if (this.environment.getPropertySources().contains("ApolloBootstrapPropertySources")) {
          this.ensureBootstrapPropertyPrecedence(this.environment);// 过滤掉bootstrap时已添加的配置，避免覆盖
          this.environment.getPropertySources().addAfter("ApolloBootstrapPropertySources", composite);
      } else {
        	// 加到头部（但此时Bean已初始化）
          this.environment.getPropertySources().addFirst(composite);
      }
  }
}
```

## 热更新

### 监听器

```java
//bootstrap方式，在com.ctrip.framework.apollo.spi.DefaultConfigFactory#create中
@Override
public Config create(String namespace) {
  ConfigFileFormat format = determineFileFormat(namespace);
  if (ConfigFileFormat.isPropertiesCompatible(format)) {
    return new DefaultConfig(namespace, createPropertiesCompatibleFileConfigRepository(namespace, format));
  }
  return new DefaultConfig(namespace, createLocalConfigRepository(namespace));
}
->
public DefaultConfig(String namespace, ConfigRepository configRepository) {
  m_namespace = namespace;
  m_resourceProperties = loadFromResource(m_namespace);
  m_configRepository = configRepository;
  m_configProperties = new AtomicReference<>();
  m_warnLogRateLimiter = RateLimiter.create(0.017); // 1 warning log output per minute
  initialize();// 注册监听器
}
```

```java
//非bootsrtp方式，在PropertySourcesProcessor#initializeAutoUpdatePropertiesFeature中
private void initializeAutoUpdatePropertiesFeature(ConfigurableListableBeanFactory beanFactory) {
  if (!configUtil.isAutoUpdateInjectedSpringPropertiesEnabled() ||
      !AUTO_UPDATE_INITIALIZED_BEAN_FACTORIES.add(beanFactory)) {
    return;
  }

  AutoUpdateConfigChangeListener autoUpdateConfigChangeListener = new AutoUpdateConfigChangeListener(
      environment, beanFactory);

  List<ConfigPropertySource> configPropertySources = configPropertySourceFactory.getAllConfigPropertySources();
  for (ConfigPropertySource configPropertySource : configPropertySources) {
    configPropertySource.addChangeListener(autoUpdateConfigChangeListener);// 注册监听器
  }
}
```

### 刷新逻辑

```java
//bootstrap方式，通过cachedThreadpool，coresize = 0，maximumPoolSize = Integer.MAX_VALUE执行
m_executorService.submit(new Runnable() {
  @Override
  public void run() {
    String listenerName = listener.getClass().getName();
    Transaction transaction = Tracer.newTransaction("Apollo.ConfigChangeListener", listenerName);
    try {
      listener.onChange(changeEvent);// 执行实际刷新逻辑
      transaction.setStatus(Transaction.SUCCESS);
    } catch (Throwable ex) {
      transaction.setStatus(ex);
      Tracer.logError(ex);
      logger.error("Failed to invoke config change listener {}", listenerName, ex);
    } finally {
      transaction.complete();
    }
  }
});
```



```java
//非bootstrap方式，通过反射直接刷新
@Override
public void onChange(ConfigChangeEvent changeEvent) {
  Set<String> keys = changeEvent.changedKeys();
  if (CollectionUtils.isEmpty(keys)) {
    return;
  }
  for (String key : keys) {
    // 1. check whether the changed key is relevant
    Collection<SpringValue> targetValues = springValueRegistry.get(beanFactory, key);
    if (targetValues == null || targetValues.isEmpty()) {
      continue;
    }

    // 2. update the value
    for (SpringValue val : targetValues) {
      updateSpringValue(val);// 反射注入，设置字段或者invoke方法
    }
  }
}
```

