
## 重点的回调机制
```
//需要配置再META-INF/spring.factories
ApplicationContextInitializer
ApplicationRunListener
//添加到ioc容器中
ApplicationRunner
CommandLineRunner
```


## 启动原理
```
    //从run方法中启动
    SpringApplication.run(SpringMain.class, args);
    
    //进入run方法发现会首先新建SpringApplication对象，再运行run方法
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return (new SpringApplication(primarySources)).run(args);
    }
    
    public SpringApplication(ResourceLoader resourceLoader, Class... primarySources) {
        this.sources = new LinkedHashSet();
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = new HashSet();
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        //将主配置类保存到primarySources
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        //判断是否是web应用，以及是reactive还是servlet
        this.webApplicationType = this.deduceWebApplicationType();
        //通过loadSpringFactories从META-INF/spring.factories找到所有配置的ApplicationContextInitializer
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //同理从META-INF/spring.factories找到所有配置的ApplicationListener
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        /**
         *例如spring-boot-autoconfigure包下包含的spring.factories
         *org.springframework.context.ApplicationContextInitializer
         *org.springframework.context.ApplicationListener
         */
        
        //从多个配置类中，找到有main方法的主配置类
        this.mainApplicationClass = this.deduceMainApplicationClass();
        //到此，spring对象创建完毕
    }
```


## 运行流程
```
//创建完spring对象后，则执行对应的run方法
    public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        //创建了一个空的IOC容器
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        //awt应用相关
        this.configureHeadlessProperty();
        //在方法中通过getSpringFactoriesInstances，从类路径下的从META-INF/spring.factories中找到对应的Listeners
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        //回调所有的SpringApplicationRunListeners的starting方法
        listeners.starting();

        Collection exceptionReporters;
        try {
            //封装命令行参数args
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //创建环境
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            /**
             *创建/获取环境，
             *回调SpringApplicationRunListeners的environmentPrepared
             *判断是否web环境，不是则会按需转换成StandardEnvironment
             *环境准备完成
             */
            
            this.configureIgnoreBeanInfo(environment);
            //printBanner则会在console开始打印springboot图标
            Banner printedBanner = this.printBanner(environment);
            //通过webApplicationType来判断创建SERVLET，REACTIVE及一般ioc容器
            context = this.createApplicationContext();
            //异常分析报告，可再catch中找到对应处理
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            
            /**
             *准备上下文环境，将environment保存到ioc容器
             *通过applyInitializers将创建spring时获取到的Initializers
             *回调所有ApplicationContextInitializer的initialize方法
             *回调所有ApplicationRunListener的contextPrepared方法
             *等到prepareContext即将完成时，会回调ApplicationRunListener的contextLoaded方法
             */
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //刷新容器，即初始化ioc容器--扫描，加载所有配置类，@Bean等
            this.refreshContext(context);
            //afterRefresh方法体内部为空了，下文中有callRunners逻辑
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }
            //回调所有SpringApplicationRunListener的started方法
            listeners.started(context);
            //callRunners来获取ioc容器中的ApplicationRunner和CommandLineRunner
            //ApplicationRunner优先级会高于CommandLineRunner
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            //回调所有SpringApplicationRunListener的running方法
            listeners.running(context);
            //springboot启动完成，会返回ioc容器
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
    
    //SpringApplicationRunListener接口中的方法
    public interface SpringApplicationRunListener {
        void starting();
    
        void environmentPrepared(ConfigurableEnvironment environment);
    
        void contextPrepared(ConfigurableApplicationContext context);
    
        void contextLoaded(ConfigurableApplicationContext context);
    
        void started(ConfigurableApplicationContext context);
    
        void running(ConfigurableApplicationContext context);
    
        void failed(ConfigurableApplicationContext context, Throwable exception);
    }
```


## 自动配置原理
```
1, @SpringBootApplication -> @EnableAutoConfiguration
2, @EnableAutoConfiguration -> @Import({AutoConfigurationImportSelector.class})
    //通过选择器来给容器中导入对应的组件
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
            AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            /**
             *通过getCandidateConfigurations->loadFactoryNames->loadSpringFactories
             *扫描类路径下所有jar包的META-INF/spring.factories
             *把扫描到的内容包装成properties文件，从properties中获取EnableAutoConfiguration的值加载到容器中
             *spring.factories -> org.springframework.boot.autoconfigure.EnableAutoConfiguration
             */
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.filter(configurations, autoConfigurationMetadata);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return StringUtils.toStringArray(configurations);
        }
    }
```
3, 加载自动配置
```
//以HttpEncodingAutoConfiguration为例
@Configuration //容器中的配置类
@EnableConfigurationProperties({HttpEncodingProperties.class}) //指定启动类的配置类{HttpEncodingProperties.class
//指定条件下配置类生效
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@ConditionalOnClass({CharacterEncodingFilter.class})
@ConditionalOnProperty(
    prefix = "spring.http.encoding",
    value = {"enabled"},
    matchIfMissing = true
)
public class HttpEncodingAutoConfiguration {}

@ConfigurationProperties(
    prefix = "spring.http.encoding"
) //从配置文件中获取对应的值和bean的属性进行绑定
public class HttpEncodingProperties {}

```



## 自定义starter
1.需要用到什么依赖;

2.如何编写自动配置；
```
@Configuration //指定这是一个配置类
@ConditionalOnXXX //在满足条件时才自动加载，例如ConditionalOnWebApplication
@AutoConfigureAfter //指定自动配置类的顺序
@Bean //给容器中添加组件

/**
 *结合
 */ConfigurationProperties和EnableConfigurationProperties来让Properties类生效并加入到容器中
@EnableConfigurationProperties({WebMvcProperties.class, ResourceProperties.class})
@ConfigurationProperties(
    prefix = "spring.mvc"
)
//最后，需要将需要加载的类配置再META-INF/spring.factories中
```
3.模式：
```
启动器只用于依赖导入；
专门写一个自动配置模块；
启动器依赖自动配置，使用时只需要引入启动器
```

