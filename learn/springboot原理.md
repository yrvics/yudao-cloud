# Spring Boot 原理深度剖析

我来从源码级别详细剖析Spring Boot的核心原理，让你真正理解它为什么能"自动配置"。

## 1. Spring Boot 的核心设计思想

### 1.1 从传统Spring到Spring Boot

```java
// 传统Spring MVC配置（繁琐）
@Configuration
@ComponentScan("com.example")
@EnableWebMvc
public class AppConfig {
    
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
    
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName("com.mysql.jdbc.Driver");
        ds.setUrl("jdbc:mysql://localhost:3306/test");
        ds.setUsername("root");
        ds.setPassword("123456");
        return ds;
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    // 还需要配置事务管理器、静态资源处理、拦截器...
}

// Spring Boot 只需要一行
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 1.2 核心设计原则

```
约定优于配置 (Convention over Configuration)
    ↓
自动配置 (Auto Configuration)
    ↓
起步依赖 (Starter Dependencies)
    ↓
内嵌容器 (Embedded Containers)
```

## 2. @SpringBootApplication 注解解析

### 2.1 源码分析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration        // 1. 配置类标志
@EnableAutoConfiguration         // 2. 开启自动配置
@ComponentScan(excludeFilters = { // 3. 组件扫描
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    
    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};
    
    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};
    
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};
    
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};
    
    @AliasFor(annotation = SpringBootConfiguration.class)
    boolean proxyBeanMethods() default true;
}
```

### 2.2 @SpringBootConfiguration 源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration  // 本质上就是@Configuration
public @interface SpringBootConfiguration {
    
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;
}
```

## 3. @EnableAutoConfiguration 核心原理

### 3.1 注解定义

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage  // 1. 注册主配置类所在包
@Import(AutoConfigurationImportSelector.class)  // 2. 核心：导入自动配置
public @interface EnableAutoConfiguration {
    
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
    Class<?>[] exclude() default {};
    
    String[] excludeName() default {};
}
```

### 3.2 @AutoConfigurationPackage 源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)  // 注册包名
public @interface AutoConfigurationPackage {
    
    String[] basePackages() default {};
    
    Class<?>[] basePackageClasses() default {};
}

// 注册器实现
static class Registrar implements ImportBeanDefinitionRegistrar {
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                        BeanDefinitionRegistry registry) {
        // 获取主配置类所在的包名
        String[] basePackages = getBasePackages(metadata);
        // 注册到BeanFactory，供后续自动配置使用
        AutoConfigurationPackages.register(registry, basePackages);
    }
}
```

### 3.3 AutoConfigurationImportSelector 完整源码解析

```java
public class AutoConfigurationImportSelector 
    implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
               BeanFactoryAware, EnvironmentAware, Ordered {
    
    // ========== 1. 核心方法：selectImports ==========
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // 检查自动配置是否启用
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        // 获取自动配置条目
        AutoConfigurationEntry autoConfigurationEntry = 
            getAutoConfigurationEntry(annotationMetadata);
        // 返回自动配置类数组
        return StringUtils.toStringArray(
            autoConfigurationEntry.getConfigurations());
    }
    
    // ========== 2. 获取自动配置条目 ==========
    protected AutoConfigurationEntry getAutoConfigurationEntry(
            AnnotationMetadata annotationMetadata) {
        
        // 检查是否启用
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        
        // 获取注解属性（exclude, excludeName等）
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        
        // 从META-INF/spring.factories和AutoConfiguration.imports获取候选配置类
        List<String> configurations = getCandidateConfigurations(
            annotationMetadata, attributes);
        
        // 去重
        configurations = removeDuplicates(configurations);
        
        // 处理排除项
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        
        // 应用条件过滤（@Conditional系列注解）
        configurations = getConfigurationClassFilter().filter(configurations);
        
        // 发送自动配置导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        
        return new AutoConfigurationEntry(configurations, exclusions);
    }
    
    // ========== 3. 获取候选配置类（核心） ==========
    protected List<String> getCandidateConfigurations(
            AnnotationMetadata metadata, AnnotationAttributes attributes) {
        
        // 从META-INF/spring.factories加载（Spring Boot 2.7之前的主要方式）
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        
        // 从META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports加载（2.7新方式）
        ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
            .forEach(configurations::add);
        
        Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories nor in " +
            "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
        
        return configurations;
    }
    
    // ========== 4. 条件过滤 ==========
    private ConfigurationClassFilter getConfigurationClassFilter() {
        if (this.configurationClassFilter == null) {
            // 获取所有ConditionEvaluator
            List<AutoConfigurationImportFilter> filters = 
                SpringFactoriesLoader.loadFactoryNames(
                    AutoConfigurationImportFilter.class, this.beanClassLoader)
                    .stream()
                    .map(this::instantiateFilter)
                    .collect(Collectors.toList());
            
            this.configurationClassFilter = new ConfigurationClassFilter(
                this.beanClassLoader, filters);
        }
        return this.configurationClassFilter;
    }
    
    // 内部类：条件过滤
    private static class ConfigurationClassFilter {
        private final AutoConfigurationImportFilter[] filters;
        
        List<String> filter(List<String> configurations) {
            // 使用过滤器（如OnClassCondition, OnWebApplicationCondition等）
            String[] candidates = StringUtils.toStringArray(configurations);
            boolean[] filtered = this.filters[0].match(candidates, this.beanClassLoader);
            
            List<String> result = new ArrayList<>();
            for (int i = 0; i < filtered.length; i++) {
                if (filtered[i]) {
                    result.add(candidates[i]);
                }
            }
            return result;
        }
    }
}
```

## 4. SpringFactoriesLoader 原理

### 4.1 加载机制

```java
public final class SpringFactoriesLoader {
    
    // 缓存所有加载的工厂
    private static final Map<ClassLoader, MultiValueMap<String, String>> cache = 
        new ConcurrentReferenceHashMap<>();
    
    // 核心加载方法
    public static List<String> loadFactoryNames(Class<?> factoryType, 
                                                  ClassLoader classLoader) {
        String factoryTypeName = factoryType.getName();
        // 从缓存或文件系统加载
        return loadSpringFactories(classLoader)
            .getOrDefault(factoryTypeName, Collections.emptyList());
    }
    
    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }
        
        result = new HashMap<>();
        try {
            // 加载所有 META-INF/spring.factories 文件
            Enumeration<URL> urls = classLoader.getResources(
                FACTORIES_RESOURCE_LOCATION); // "META-INF/spring.factories"
            
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(
                    new UrlResource(url));
                
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    String factoryTypeName = ((String) entry.getKey()).trim();
                    String[] factoryImplementationNames = 
                        StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                    
                    for (String implName : factoryImplementationNames) {
                        result.computeIfAbsent(factoryTypeName, 
                            key -> new ArrayList<>()).add(implName.trim());
                    }
                }
            }
            
            cache.put(classLoader, result);
            return result;
        } catch (IOException ex) {
            throw new IllegalArgumentException(
                "Unable to load factories from location [" + 
                FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
    }
}
```

### 4.2 spring.factories 文件示例

```properties
# META-INF/spring.factories
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener

# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer

# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.diagnostics.analyzer.BeanNotOfRequiredTypeFailureAnalyzer
```

## 5. 条件注解体系详解

### 5.1 @Conditional 核心接口

```java
@FunctionalInterface
public interface Condition {
    // 判断条件是否满足
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}

public class ConditionContext {
    private final ConfigurableListableBeanFactory beanFactory;
    private final ClassLoader classLoader;
    private final Environment environment;
    private final ResourceLoader resourceLoader;
    private final BeanDefinitionRegistry registry;
    // getters...
}
```

### 5.2 @ConditionalOnClass 实现

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)  // 指定条件类
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
    String[] name() default {};
}

// 条件类实现
public class OnClassCondition extends SpringBootCondition {
    
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                            AnnotatedTypeMetadata metadata) {
        // 获取注解属性
        ClassLoader classLoader = context.getClassLoader();
        
        // 获取value和name属性
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        List<String> missing = new ArrayList<>();
        
        // 检查每个类是否存在
        for (String className : onClasses) {
            if (!isPresent(className, classLoader)) {
                missing.add(className);
            }
        }
        
        if (!missing.isEmpty()) {
            return ConditionOutcome.noMatch(
                "Class not found: " + missing);
        }
        
        return ConditionOutcome.match();
    }
    
    private boolean isPresent(String className, ClassLoader classLoader) {
        try {
            Class.forName(className, false, classLoader);
            return true;
        } catch (ClassNotFoundException ex) {
            return false;
        }
    }
}
```

### 5.3 @ConditionalOnMissingBean 实现

```java
public class OnBeanCondition extends SpringBootCondition {
    
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                            AnnotatedTypeMetadata metadata) {
        // 获取注解属性
        MultiValueMap<String, Object> attributes = 
            metadata.getAllAnnotationAttributes(
                ConditionalOnMissingBean.class.getName(), true);
        
        // 检查Bean是否存在
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        String[] beanNames = beanFactory.getBeanNamesForType(type);
        
        if (beanNames.length > 0) {
            return ConditionOutcome.noMatch(
                "Bean already exists: " + Arrays.toString(beanNames));
        }
        
        return ConditionOutcome.match();
    }
}
```

### 5.4 @ConditionalOnProperty 实现

```java
public class OnPropertyCondition extends SpringBootCondition {
    
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                            AnnotatedTypeMetadata metadata) {
        // 获取注解属性
        Map<String, Object> attributes = metadata.getAnnotationAttributes(
            ConditionalOnProperty.class.getName());
        
        String prefix = (String) attributes.get("prefix");
        String name = (String) attributes.get("name");
        String havingValue = (String) attributes.get("havingValue");
        boolean matchIfMissing = (boolean) attributes.get("matchIfMissing");
        
        // 获取配置值
        Environment environment = context.getEnvironment();
        String propertyName = prefix != null ? prefix + "." + name : name;
        String propertyValue = environment.getProperty(propertyName);
        
        if (propertyValue == null) {
            return ConditionOutcome.match(matchIfMissing, 
                "Property not found, matchIfMissing=" + matchIfMissing);
        }
        
        // 检查值是否匹配
        if (havingValue != null && !havingValue.isEmpty()) {
            boolean match = propertyValue.equals(havingValue);
            return new ConditionOutcome(match, 
                "Property value '" + propertyValue + "' " + 
                (match ? "matches" : "does not match") + " '" + havingValue + "'");
        }
        
        return ConditionOutcome.match("Property found");
    }
}
```

## 6. 自动配置类示例详解

### 6.1 DataSourceAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, 
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {
    
    // 配置1：嵌入式数据库（H2、HSQL、Derby）
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnEmbeddedDatabase  // 只有嵌入式数据库才生效
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    protected static class EmbeddedDatabaseConfiguration {
        
        @Bean
        @ConditionalOnMissingBean
        public DataSource dataSource(DataSourceProperties properties) {
            return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .setName(properties.getName())
                .build();
        }
    }
    
    // 配置2：连接池（HikariCP、Tomcat、DBCP2）
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean(DataSource.class)
    @ConditionalOnClass(name = "com.zaxxer.hikari.HikariDataSource")
    protected static class Hikari {
        
        @Bean
        @ConditionalOnMissingBean
        public DataSource dataSource(DataSourceProperties properties) {
            HikariDataSource dataSource = new HikariDataSource();
            dataSource.setJdbcUrl(properties.getUrl());
            dataSource.setUsername(properties.getUsername());
            dataSource.setPassword(properties.getPassword());
            dataSource.setDriverClassName(properties.getDriverClassName());
            return dataSource;
        }
    }
}
```

### 6.2 WebMvcAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, 
                      TaskExecutionAutoConfiguration.class, 
                      ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
    
    // 1. 配置ViewResolver
    @Bean
    @ConditionalOnMissingBean
    public InternalResourceViewResolver defaultViewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
    
    // 2. 配置静态资源处理
    @Configuration(proxyBeanMethods = false)
    @Import(EnableWebMvcConfiguration.class)
    @EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
    @Order(0)
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
        
        @Override
        public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                return;
            }
            // 配置静态资源路径
            addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
            addResourceHandler(registry, "/**", this.resourceProperties.getStaticLocations());
        }
        
        @Override
        public void configurePathMatch(PathMatchConfigurer configurer) {
            // 配置路径匹配
            if (this.mvcProperties.getPathmatch().getMatchingStrategy() == 
                WebMvcProperties.MatchingStrategy.ANT_PATH_MATCHER) {
                configurer.setPathMatcher(new AntPathMatcher());
            }
        }
        
        // 3. 配置消息转换器
        @Override
        public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
            converters.add(new MappingJackson2HttpMessageConverter());
            converters.add(new StringHttpMessageConverter());
        }
        
        // 4. 配置拦截器
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            // 配置LocaleChangeInterceptor
            registry.addInterceptor(localeChangeInterceptor);
            // 配置ThemeChangeInterceptor
            registry.addInterceptor(themeChangeInterceptor);
        }
    }
    
    // 5. 配置DispatcherServlet
    @Bean
    @ConditionalOnMissingBean
    public DispatcherServlet dispatcherServlet() {
        DispatcherServlet dispatcherServlet = new DispatcherServlet();
        dispatcherServlet.setThrowExceptionIfNoHandlerFound(true);
        return dispatcherServlet;
    }
}
```

## 7. SpringApplication 启动流程详解

### 7.1 SpringApplication.run() 完整流程

```java
public class SpringApplication {
    
    public ConfigurableApplicationContext run(String... args) {
        // 1. 启动计时器
        long startTime = System.nanoTime();
        
        // 2. 创建启动上下文
        DefaultBootstrapContext bootstrapContext = createBootstrapContext();
        ConfigurableApplicationContext context = null;
        
        // 3. 配置无头模式
        configureHeadlessProperty();
        
        // 4. 获取运行监听器
        SpringApplicationRunListeners listeners = getRunListeners(args);
        
        // 5. 发送启动事件
        listeners.starting(bootstrapContext, this.mainApplicationClass);
        
        try {
            // 6. 创建ApplicationArguments
            ApplicationArguments applicationArguments = 
                new DefaultApplicationArguments(args);
            
            // 7. 准备环境
            ConfigurableEnvironment environment = prepareEnvironment(
                listeners, bootstrapContext, applicationArguments);
            
            // 8. 配置忽略Bean信息
            configureIgnoreBeanInfo(environment);
            
            // 9. 打印Banner
            Banner printedBanner = printBanner(environment);
            
            // 10. 创建ApplicationContext
            context = createApplicationContext();
            
            // 11. 设置ApplicationContext的启动阶段
            context.setApplicationStartup(this.applicationStartup);
            
            // 12. 准备上下文
            prepareContext(bootstrapContext, context, environment, listeners, 
                applicationArguments, printedBanner);
            
            // 13. 刷新上下文（核心）
            refreshContext(context);
            
            // 14. 刷新后处理
            afterRefresh(context, applicationArguments);
            
            // 15. 计算启动时间
            Duration timeTakenToStartup = Duration.ofNanos(
                System.nanoTime() - startTime);
            
            // 16. 发送启动完成事件
            listeners.started(context, timeTakenToStartup);
            
            // 17. 调用Runner
            callRunners(context, applicationArguments);
            
            // 18. 发送运行中事件
            Duration timeTakenToReady = Duration.ofNanos(
                System.nanoTime() - startTime);
            listeners.ready(context, timeTakenToReady);
            
            return context;
        } catch (Throwable ex) {
            handleRunFailure(context, ex, listeners);
            throw new IllegalStateException(ex);
        }
    }
    
    // 创建ApplicationContext
    protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                switch (this.webApplicationType) {
                    case SERVLET:
                        // AnnotationConfigServletWebServerApplicationContext
                        contextClass = Class.forName(
                            DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                        break;
                    case REACTIVE:
                        // AnnotationConfigReactiveWebServerApplicationContext
                        contextClass = Class.forName(
                            DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                        break;
                    default:
                        // AnnotationConfigApplicationContext
                        contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
                }
            } catch (ClassNotFoundException ex) {
                throw new IllegalStateException(ex);
            }
        }
        return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
}
```

### 7.2 refreshContext 核心流程

```java
// AbstractApplicationContext.refresh() 源码
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
        
        // 1. 准备刷新
        prepareRefresh();
        
        // 2. 获取BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        // 3. 准备BeanFactory
        prepareBeanFactory(beanFactory);
        
        try {
            // 4. 允许子类后处理BeanFactory
            postProcessBeanFactory(beanFactory);
            
            StartupStep beanPostProcess = this.applicationStartup.start(
                "spring.context.beans.post-process");
            
            // 5. 调用BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);
            
            // 6. 注册BeanPostProcessor
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();
            
            // 7. 初始化MessageSource
            initMessageSource();
            
            // 8. 初始化事件广播器
            initApplicationEventMulticaster();
            
            // 9. 初始化特定子类（如WebServer）
            onRefresh();
            
            // 10. 注册监听器
            registerListeners();
            
            // 11. 初始化所有单例Bean（懒加载除外）
            finishBeanFactoryInitialization(beanFactory);
            
            // 12. 完成刷新（启动WebServer、发布事件等）
            finishRefresh();
        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
            }
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}

// finishBeanFactoryInitialization - 实例化单例Bean
protected void finishBeanFactoryInitialization(
        ConfigurableListableBeanFactory beanFactory) {
    
    // 初始化ConversionService
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }
    
    // 注册默认的EmbeddedValueResolver
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(
            strVal -> getEnvironment().resolvePlaceholders(strVal));
    }
    
    // 初始化LoadTimeWeaverAware
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(
        LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }
    
    // 冻结配置
    beanFactory.freezeConfiguration();
    
    // 实例化所有非懒加载单例Bean（核心）
    beanFactory.preInstantiateSingletons();
}
```

### 7.3 preInstantiateSingletons 源码

```java
// DefaultListableBeanFactory
@Override
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }
    
    // 获取所有Bean名称
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
    
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非抽象、单例、非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 如果是FactoryBean
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(
                            (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                            getAccessControlContext());
                    } else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            } else {
                // 普通Bean，直接实例化
                getBean(beanName);
            }
        }
    }
    
    // 触发所有SmartInitializingSingleton的回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            StartupStep smartInitialize = this.getApplicationStartup().start(
                "spring.beans.smart-initialize")
                .tag("beanName", beanName);
            SmartInitializingSingleton smartSingleton = 
                (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            } else {
                smartSingleton.afterSingletonsInstantiated();
            }
            smartInitialize.end();
        }
    }
}
```

## 8. 内嵌Web服务器启动原理

### 8.1 ServletWebServerApplicationContext

```java
public class ServletWebServerApplicationContext 
    extends GenericWebApplicationContext
    implements ConfigurableWebServerApplicationContext {
    
    private volatile WebServer webServer;
    private ServletConfig servletConfig;
    private ServletContext servletContext;
    
    // 刷新时创建WebServer
    @Override
    protected void onRefresh() {
        super.onRefresh();
        try {
            createWebServer();
        } catch (Throwable ex) {
            throw new ApplicationContextException("Unable to start web server", ex);
        }
    }
    
    private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = getServletContext();
        if (webServer == null && servletContext == null) {
            // 获取ServletWebServerFactory
            ServletWebServerFactory factory = getWebServerFactory();
            // 创建WebServer
            this.webServer = factory.getWebServer(getSelfInitializer());
            getBeanFactory().registerSingleton("webServerGracefulShutdown",
                new WebServerGracefulShutdownLifecycle(this.webServer));
            getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this, this.webServer));
        } else if (servletContext != null) {
            try {
                getSelfInitializer().onStartup(servletContext);
            } catch (ServletException ex) {
                throw new ApplicationContextException("Cannot initialize servlet context", ex);
            }
        }
        initPropertySources();
    }
    
    // 完成刷新时启动WebServer
    @Override
    protected void finishRefresh() {
        super.finishRefresh();
        WebServer webServer = startWebServer();
        if (webServer != null) {
            publishEvent(new ServletWebServerInitializedEvent(webServer, this));
        }
    }
    
    private WebServer startWebServer() {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.start();
        }
        return webServer;
    }
}
```

### 8.2 TomcatServletWebServerFactory

```java
public class TomcatServletWebServerFactory 
    extends AbstractServletWebServerFactory
    implements ConfigurableTomcatWebServerFactory {
    
    @Override
    public WebServer getWebServer(ServletContextInitializer... initializers) {
        if (this.disableMBeanRegistry) {
            Registry.disableRegistry();
        }
        // 创建Tomcat实例
        Tomcat tomcat = new Tomcat();
        File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        
        // 创建连接器
        Connector connector = new Connector(this.protocol);
        connector.setThrowOnFailure(true);
        tomcat.getService().addConnector(connector);
        customizeConnector(connector);
        tomcat.setConnector(connector);
        
        // 创建Host
        tomcat.getHost().setAutoDeploy(false);
        configureEngine(tomcat.getEngine());
        
        // 创建Context
        prepareContext(tomcat.getHost(), initializers);
        
        // 返回TomcatWebServer
        return getTomcatWebServer(tomcat);
    }
    
    protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
        return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
    }
}

// TomcatWebServer启动
public class TomcatWebServer implements WebServer {
    
    private final Tomcat tomcat;
    private volatile boolean started;
    
    @Override
    public void start() throws WebServerException {
        synchronized (this.monitor) {
            if (this.started) {
                return;
            }
            try {
                addPreviouslyRemovedConnectors();
                // 启动Tomcat
                this.tomcat.start();
                // 等待连接器启动
                for (Connector connector : this.tomcat.getService().findConnectors()) {
                    if (connector.getState() != LifecycleState.STARTED) {
                        connector.start();
                    }
                }
                this.started = true;
            } catch (Exception ex) {
                throw new WebServerException("Unable to start embedded Tomcat", ex);
            }
        }
    }
}
```

## 9. Starter 原理与自定义

### 9.1 自定义Starter结构

```
my-spring-boot-starter/
├── pom.xml
└── src/main/java/
    └── com/example/
        ├── MyService.java
        ├── MyProperties.java
        ├── MyAutoConfiguration.java
        └── spring.factories
```

### 9.2 完整示例

```java
// 1. 配置属性类
@ConfigurationProperties(prefix = "my.service")
public class MyProperties {
    private boolean enabled = true;
    private String host = "localhost";
    private int port = 8080;
    private List<String> servers = new ArrayList<>();
    
    // getters and setters
}

// 2. 服务类
public class MyService {
    private final MyProperties properties;
    
    public MyService(MyProperties properties) {
        this.properties = properties;
    }
    
    public void doSomething() {
        System.out.println("MyService running with " + properties);
    }
}

// 3. 自动配置类
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "my.service", name = "enabled", 
                           havingValue = "true", matchIfMissing = true)
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
    
    @Configuration
    @ConditionalOnClass(MyService.class)
    @ConditionalOnProperty(prefix = "my.service", name = "health-enabled", 
                           havingValue = "true", matchIfMissing = false)
    public static class HealthCheckConfiguration {
        
        @Bean
        public HealthIndicator myServiceHealthIndicator(MyService myService) {
            return new MyServiceHealthIndicator(myService);
        }
    }
}

// 4. 健康检查（可选）
public class MyServiceHealthIndicator implements HealthIndicator {
    private final MyService myService;
    
    public MyServiceHealthIndicator(MyService myService) {
        this.myService = myService;
    }
    
    @Override
    public Health health() {
        try {
            // 执行健康检查
            return Health.up().withDetail("status", "OK").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

// 5. spring.factories
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration

// 6. 使用
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 10. 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Boot 启动流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  @SpringBootApplication                                          │
│       │                                                          │
│       ├── @SpringBootConfiguration                              │
│       │      └── @Configuration                                 │
│       │                                                          │
│       ├── @EnableAutoConfiguration                              │
│       │      ├── @AutoConfigurationPackage                      │
│       │      │      └── 注册主配置类包名                         │
│       │      │                                                  │
│       │      └── @Import(AutoConfigurationImportSelector)       │
│       │             │                                            │
│       │             └── selectImports()                         │
│       │                    ├── 从spring.factories加载配置类     │
│       │                    ├── 应用exclude过滤                  │
│       │                    ├── 应用@Conditional过滤              │
│       │                    └── 返回配置类数组                    │
│       │                                                          │
│       └── @ComponentScan                                        │
│              └── 扫描组件                                        │
│                                                                  │
│  SpringApplication.run()                                        │
│       │                                                          │
│       ├── 创建ApplicationContext                                │
│       │                                                          │
│       ├── refresh()                                             │
│       │      ├── invokeBeanFactoryPostProcessors()              │
│       │      │      └── ConfigurationClassPostProcessor         │
│       │      │             └── 解析@Configuration类              │
│       │      │                    ├── 处理@Import               │
│       │      │                    │      └── 调用selectImports   │
│       │      │                    ├── 处理@Bean                  │
│       │      │                    └── 处理@ComponentScan         │
│       │      │                                                  │
│       │      ├── registerBeanPostProcessors()                   │
│       │      │                                                  │
│       │      ├── initApplicationEventMulticaster()              │
│       │      │                                                  │
│       │      ├── onRefresh()                                    │
│       │      │      └── 创建WebServer (Tomcat/Jetty/Undertow)   │
│       │      │                                                  │
│       │      ├── finishBeanFactoryInitialization()              │
│       │      │      └── 实例化所有单例Bean                       │
│       │      │                                                  │
│       │      └── finishRefresh()                                │
│       │             └── 启动WebServer                           │
│       │                                                          │
│       └── 发布ApplicationReadyEvent                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 11. 核心总结

### 11.1 Spring Boot 三大核心机制

| 机制 | 实现方式 | 作用 |
|------|---------|------|
| **自动配置** | `@EnableAutoConfiguration` + `AutoConfigurationImportSelector` | 根据类路径和配置自动创建Bean |
| **条件装配** | `@Conditional` 系列注解 | 根据条件决定是否创建Bean |
| **起步依赖** | Starter POM | 统一管理依赖版本和传递性依赖 |

### 11.2 关键类职责

| 类名 | 职责 |
|------|------|
| `SpringApplication` | 启动入口，管理整个启动流程 |
| `AutoConfigurationImportSelector` | 加载并过滤自动配置类 |
| `SpringFactoriesLoader` | 加载META-INF/spring.factories中的配置 |
| `ConfigurationClassPostProcessor` | 解析@Configuration类 |
| `ConditionEvaluator` | 评估@Conditional条件 |

### 11.3 设计思想

1. **约定优于配置**：合理的默认值，减少配置量
2. **开闭原则**：对扩展开放，对修改关闭（通过条件注解）
3. **单一职责**：每个自动配置类负责特定功能
4. **依赖倒置**：通过starter统一管理依赖

这就是Spring Boot的完整原理。核心就是通过`@EnableAutoConfiguration`+`AutoConfigurationImportSelector`加载配置类，通过`@Conditional`系列注解实现条件化配置，最终实现"开箱即用"的效果。