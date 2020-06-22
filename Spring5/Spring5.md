# Spring5

## IOC 控制反转，将bean交给spring管理
### DI-基于xml的属性注入
```
    <bean id="responseStatus" class="com.yitu.entries.ResponseStatus">
        <!-- 基于setter方法注入 -->
        <property name="Id" value="0"></property>
        <property name="StatusCode" value="200"></property>

        <!-- 基于构造器注入,需要创建对应参数的构造器 -->
        <constructor-arg name="id" value="0"></constructor-arg>
        <constructor-arg name="statusCode" value="200"></constructor-arg>
    </bean>
    
    and so on..
```


### IOC实现方式
#### BeanFactory
```
Spring内部接口
创建BeanFactory对象时，只会获取对应xml文件，不会主动创建bean对象。
只有在调用getBean方法时才会创建bean对象。
```

#### ApplicationContext
```
BeanFactory的子接口，面向开发者
创建ApplicationContext时，会加载对应xml文件，且主动创建bean对象。
```

### bean的作用域
#### singleton 默认
```
单例：当加载class时，spring就为class创建好了对象
```

#### prototype
```
多例：当调用getBean方法时，才会创建对象，且每次都会创建一个新的实例
```

#### web中的特殊实例
```
request：每次请求实例化一个bean。（仅适用于WebApplicationContext环境）
session：在一次会话中共享一个bean。（仅适用于WebApplicationContext环境）
```

### bean的生命周期
```
1，通过构造器创建bean对象
2，通过set方法设置属性值
(beforeInit)
3，执行初始化方法(实现BeanPostProcessor可以控制创建bean前后执行的方法-后置处理器)
(afterInit)
4，获取bean对象
5，IOC容器关闭时调用销毁方法
```

### spring 如何解决循环依赖 (默认的单例模式下)
```
 ①构造器的循环依赖：这种依赖spring是处理不了的，spring通过singletonsCurrentlylnCreation.add(beanName）将bean放到一个标识池中，如果创建bean的过程中，发现当前bean在池中，则会直接抛出BeanCurrentlylnCreationException异常。 
 ②单例模式下的setter循环依赖：通过“三级缓存”处理循环依赖。
 /**
 *singletonObjects -一级缓存- 用于保存 beanName 和创建的 bean 实例之间的关系， beanName --> beanInstance，注入属性的bean
 *earlySingletonObjects -二级缓存- beanName 和创建 bean，未注入属性的bean
 *singletonFactories -三级缓存- beanName 和创建bean 的工厂 之间的关系， beanName -> ObjectFactory实例之间的关系,当bean还在创建过程中，就可以通过getBean获取
 *registerSingletons -- 保存当前所有已注册的 bean
 */
 
 Spring在创建循环依赖的bean A&B的时候，会从上述三级缓存中获取。
 1.(getSingleton)如果此时beanA还在创建过程中，则会按照一->二->三级缓存的顺序，最后从三级缓存中获取到还未注入属性的beanA，然后将beanA从三级缓存移至二级缓存；
 2.(addSingletonFactory)如果当前beanA不在singletonObjects中，即还未完成注入，则将当前beanA的单例工厂(singletonFactory)放入三级缓存，移除当前beanA的二级缓存；
 3.(populateBean)beanA开始注入属性，发现依赖beanB，则去getBean(B),发现没有beanB则会创建beanB。
 在循环上述过程中，发现beanB依赖 beanA，则去getBean(A).然后发现可以直接从第三级缓存中通过singletonFactories直接获取到完成初始化但还未注入属性的beanA。
 后续则按逻辑完成beanB的属性注入后，将beanB放入一级缓存。此时逻辑回到beanA，可以获取到beanB则也完成了属性注入，也会将beanA放入一级缓存。
 以上，则解决了循环依赖的问题。
 
 ③非单例循环依赖：Spring无法对prototype的bean完成依赖注入，所以容器不会缓存prototype的bean.
```
#### 以下代码基于SpringBoot 2.3.1.RELEASE
#### 1.通过启动Spring初始化容器，初始化上下文ApplicationContext
```
    ApplicationContext ac = new AnnotationConfigApplicationContext(BeanConfig.class);
```

#### 2.进入AbstractApplicationContext.refresh
```
        public void refresh() throws BeansException, IllegalStateException {
        //同步锁
        synchronized(this.startupShutdownMonitor) {
            //重置earlyApplicationListeners，applicationListeners
            this.prepareRefresh();
            //通过refreshBeanFactory销毁之前的BeanFactory
            //通过getBeanFactory创建新的BeanFactory
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            //设置BeanFactory的各种属性
            this.prepareBeanFactory(beanFactory);

            try {
                //添加一个BeanPostProcessor到BeanFactory
                //通过WebApplicationContextUtils给BeanFactory注册servletContext
                //作用是让BeanFactory能获取到servletContext
                this.postProcessBeanFactory(beanFactory);
                //对BeanFactoryPostProcessors的一系列操作，不知在干啥
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                //初始化信息源-- 
                //判断工厂中messageSource对象，有则获取，无则创
                this.initMessageSource();
                //初始化程序时间多播器 -- 
                //判断工厂中applicationEventMulticaster对象，有则获取，无则创建
                this.initApplicationEventMulticaster();
                //判断工厂中themeSource对象
                this.onRefresh();
                //给工厂中的applicationEventMulticaster对象加上程序监听
                this.registerListeners();
                //在beanFactory.preInstantiateSingletons()方法中:
                //迭代扫描到的所有Bean
                //按照获取到的Bean的Name进入AbstractBeanFactory.doGetBean
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```
#### 3.进入AbstractBeanFactory.doGetBean
```
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
        String beanName = this.transformedBeanName(name);
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        //判断Spring的三级缓存中是否有对应的bean
        if (sharedInstance != null && args == null) {
            if (this.logger.isTraceEnabled()) {
                if (this.isSingletonCurrentlyInCreation(beanName)) {
                    this.logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    this.logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            //如果存在，则直接根据beanName返回对应的bean
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } 
        
        else {
            //如果是Prototype的bean，则会抛出BeanCurrentlyInCreationException
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                }

                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                if (requiredType != null) {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }

                return parentBeanFactory.getBean(nameToLookup);
            }

            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
                RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
                this.checkMergedBeanDefinition(mbd, beanName, args);
                String[] dependsOn = mbd.getDependsOn();
                String[] var11;
                if (dependsOn != null) {
                    var11 = dependsOn;
                    int var12 = dependsOn.length;

                    for(int var13 = 0; var13 < var12; ++var13) {
                        String dep = var11[var13];
                        if (this.isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }

                        this.registerDependentBean(dep, beanName);

                        try {
                            this.getBean(dep);
                        } catch (NoSuchBeanDefinitionException var24) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "'" + beanName + "' depends on missing bean '" + dep + "'", var24);
                        }
                    }
                }
                //单例模式，则getSingleton
                if (mbd.isSingleton()) {
                    sharedInstance = this.getSingleton(beanName, () -> {
                        try {
                            //进入AbstractAutowireCapableBeanFactory.createBean
                            return this.createBean(beanName, mbd, args);
                        } catch (BeansException var5) {
                            this.destroySingleton(beanName);
                            throw var5;
                        }
                    });
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                } else if (mbd.isPrototype()) {
                    var11 = null;

                    Object prototypeInstance;
                    try {
                        this.beforePrototypeCreation(beanName);
                        prototypeInstance = this.createBean(beanName, mbd, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
                    String scopeName = mbd.getScope();
                    Scope scope = (Scope)this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }

                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            this.beforePrototypeCreation(beanName);

                            Object var4;
                            try {
                                var4 = this.createBean(beanName, mbd, args);
                            } finally {
                                this.afterPrototypeCreation(beanName);
                            }

                            return var4;
                        });
                        bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    } catch (IllegalStateException var23) {
                        throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var23);
                    }
                }
            } catch (BeansException var26) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var26;
            }
        }

        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                } else {
                    return convertedBean;
                }
            } catch (TypeMismatchException var25) {
                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var25);
                }

                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        } else {
            return bean;
        }
    }
```
#### 4.在Spring的三级缓存中查找Bean
```
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //首先在singletonObjects中查找bean
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
        synchronized(this.singletonObjects) {
            //在earlySingletonObjects查找bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                //在singletonFactories中查找bean
                ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    //在singletonFactories中找到bean则
                    //1.将bean放入earlySingletonObjects中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    //在singletonFactories中移除已经找到的bean
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    //在缓存中找到则返回bean，未找到则返回null
    return singletonObject;
}
```
#### 5.没有在缓存中找到的bean则进入AbstractAutowireCapableBeanFactory.createBean
```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Creating instance of bean '" + beanName + "'");
        }

        RootBeanDefinition mbdToUse = mbd;
        Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        try {
            mbdToUse.prepareMethodOverrides();
        } catch (BeanDefinitionValidationException var9) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var9);
        }

        Object beanInstance;
        try {
            //前置处理器resolveBeforeInstantiation,实现apo，事务，bean
            beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
            if (beanInstance != null) {
                return beanInstance;
            }
        } catch (Throwable var10) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var10);
        }

        try {
            //doCreateBean实际创建bean
            beanInstance = this.doCreateBean(beanName, mbdToUse, args);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Finished creating instance of bean '" + beanName + "'");
            }

            return beanInstance;
        } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
            throw var7;
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
        }
    }
```
#### 6.doCreateBean中创建bean，实例化bean 跟 注入属性这两步分开。
```
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }

    if (instanceWrapper == null) {
        //根据策略创建bean实例 -- factory & Constructor
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }
    //通过构造器生成还没有注入属性的bean
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    synchronized(mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable var17) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
            }

            mbd.postProcessed = true;
        }
    }
    
    //通过单例 && 允许循环依赖 && 单例在创建中判断是否需要提前暴露
    boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
    if (earlySingletonExposure) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
        }
        
        //循环依赖解决核心：（单例模式下解决循环依赖都是需要提早曝光的）
        //通过singletonFactories把 ObjectFactory 加入到 singletonObjects中
        //移除earlySingletonObjects中的bean
        //将bean放入registeredSingletons
        this.addSingletonFactory(beanName, () -> {
            return this.getEarlyBeanReference(beanName, mbd, bean);
        });
    }

    Object exposedObject = bean;

    try {
        //开始注入属性
        this.populateBean(beanName, mbd, instanceWrapper);
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable var18) {
        if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
            throw (BeanCreationException)var18;
        }

        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = this.getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                String[] dependentBeans = this.getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                String[] var12 = dependentBeans;
                int var13 = dependentBeans.length;

                for(int var14 = 0; var14 < var13; ++var14) {
                    String dependentBean = var12[var14];
                    if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }

                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    try {
        this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
        return exposedObject;
    } catch (BeanDefinitionValidationException var16) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
    }
}
```







## AOP 面向切面，不修改源代码的情况下进行功能增强 -- 动态代理
### AOP相关术语
#### 连接点 ： 类中可以被增强的方法
#### 切入点 ： 实际被增强的方法  @Aspect
```
切入点表达式
execution([权限修饰符][返回类型][类全路径][方法名称]([参数列表])
execution(* com.eric.dao.UserDao.add(String name))
execution(* com.eric.dao.UserDao.*(String name))
execution(* com.eric.dao.*.*(String name))
```

#### 通知(增强) ： 实际增强的逻辑部分
```
前置通知 -- before
后置通知 -- after
环绕通知 -- around(before & after)
异常通知 -- catch
最终通知 -- finally

around(before)->before->function->around(after)->after->afterReturning
catch在抛出异常时通知 -- afterRetuning在抛出异常后不执行，after会先于异常执行
```
#### 切面 ： 把通知应用到切入点的过程  @PointCut

#### jdk动态代理(通过接口代理对象)
```
public class Main {
    public static void main(String[] args) {
        UserDao user1 = (UserDao) Proxy.newProxyInstance(UserDaoImpl.class.getClassLoader(), UserDaoImpl.class.getInterfaces(), new InvocationHandlerImpl(new UserDaoImpl()));
        user1.add("test");
    }
}

class InvocationHandlerImpl implements InvocationHandler {
    private Object object;
    public InvocationHandlerImpl(Object object){
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before : " + method.getName());
        Object res = method.invoke(object, args);
        System.out.println("after : " + res);
        return res;
    }
}
```

#### cglib动态代理(通过子类代理对象)


## 事务
### 事物的特性 ACID
```
1，原子性  --  操作不可分割
2，一致性  --  操作前后总量不变
3，隔离性  --  多事务间不会互相影响
4，持久性  --  持久的更改了数据
```
### Spring的事务管理器 PlatformTransactionManager interface

### @Transactional 注解
```
1. propagation : 事务的传播行为
    required：有事务则在当前事务中运行，没有则开启一个新事务
    required_new：开启一个新的事务
    supports：有事务则在当前事务中运行，否则可以不开启事务
    not_supported：当前方法不应该运行在事务中，如果有则将事务挂起
    mandatory：当前方法必须在事务中运行，如果没有事务则抛出异常
    never：当前方法不应该运行在事务中，如果有则抛出异常
    nested：如果有则在当前事务的嵌套事务中运行，否则启动新事务，并在自己的事务中运行

2. ioslation : 事务的隔离级别
    读未提交，读已提交(oracle)，可重复读(mysql)，串行
    面对的问题：
    脏读：未提交事务读取到其他未提交事务中的数据
    不可重复读：未提交事务读取到其他提交事务中修改的数据
    幻读：未提交事务读取到其他提交事务中添加的数据

    
3. timeout : 超时时间
    事务需要在一定时间内提交(默认-1)，不提交则回滚
4. readOnly : 是否只读
    只可以查询(false)
5. rollbackFor : 回滚
    设置出现哪些异常则进行回滚
6. noRollbackFor : 不回滚
    设置出现哪些异常则不进行回滚
```
