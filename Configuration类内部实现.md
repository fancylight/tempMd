# 概述 
spring-aop中提供了通过配置类替代xml的配置方式,此处讨论`Configuration`
## 大致流程(ConfigurationClassPostProcessor)
```flow
st=>start: 扫描处理
ed=>end: 结束
op1=>operation: #postProcessBeanDefinitionRegistry(注册bd)
op2=>operation: #enhanceConfigurationClasses(加强)
op3=>operation: ConfigurationClassEnhancer#enhance(cglib代理)
op4=>operation: 改变符合条件bd内部的beanClass
st->op1->op2->op3->op4->ed
```
在第一步中会将定义为`@Bean`创建为调用对应`Config#method`的工厂bd,会在`context#refresh`被按个创建
### @Configuration增强逻辑
`ConfigurationClassEnhancer`:该类内部实际创建了一个新的代理对象
```java

	public Class<?> enhance(Class<?> configClass, @Nullable ClassLoader classLoader) {
		if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
			return configClass;
		}
        //核心逻辑
		Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
		if (logger.isTraceEnabled()) {
			logger.trace(String.format("Successfully enhanced %s; enhanced class name is: %s",
					configClass.getName(), enhancedClass.getName()));
		}
		return enhancedClass;
	}
    //cglib在spring内部直接fork了一份
    private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
		Enhancer enhancer = new Enhancer();
        //需要代理的class
		enhancer.setSuperclass(configSuperClass);
        //新类的接口
		enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
		enhancer.setUseFactory(false);
        //命名方法
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setAttemptLoad(true);
        //策略,内部给新类加了一个属性
		enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
        //真正的代理逻辑
		enhancer.setCallbackFilter(CALLBACK_FILTER);
		enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
		return enhancer;
	}
    //两个逻辑
	static final Callback[] CALLBACKS = new Callback[] {
			new BeanMethodInterceptor(),
			new BeanFactoryAwareMethodInterceptor(),
			NoOp.INSTANCE
	};    
```
简述代理的每部分功能:
`EnhancedConfiguration`:继承了`BeanFactoryAware`,一个标记接口,就是为了给代理的对象内部设置`bf`
`SpringNamingPolicy`:设置最终cglib对象的名称
`BeanFactoryAwareGeneratorStrategy`:给代理对象内部加入(ARM)了一个名为`$$beanFactory`的`BeanFactory`属性
```java
	private static class BeanFactoryAwareMethodInterceptor implements MethodInterceptor, ConditionalCallback {
        //给代理对象内部设置属性
		@Override
		@Nullable
		public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
			Field field = ReflectionUtils.findField(obj.getClass(), BEAN_FACTORY_FIELD);
			Assert.state(field != null, "Unable to find generated BeanFactory field");
			field.set(obj, args[0]);

			// Does the actual (non-CGLIB) superclass implement BeanFactoryAware?
			// If so, call its setBeanFactory() method. If not, just exit.
			if (BeanFactoryAware.class.isAssignableFrom(ClassUtils.getUserClass(obj.getClass().getSuperclass()))) {
				return proxy.invokeSuper(obj, args);
			}
			return null;
		}

		@Override
		public boolean isMatch(Method candidateMethod) {
			return isSetBeanFactory(candidateMethod);
		}
        //特别对函数做了校验,保证只有是setFactory函数调用时匹配
		public static boolean isSetBeanFactory(Method candidateMethod) {
			return (candidateMethod.getName().equals("setBeanFactory") &&
					candidateMethod.getParameterCount() == 1 &&
					BeanFactory.class == candidateMethod.getParameterTypes()[0] &&
					BeanFactoryAware.class.isAssignableFrom(candidateMethod.getDeclaringClass()));
		}
	}
```
真正的调用逻辑
```java
private static class BeanMethodInterceptor implements MethodInterceptor, ConditionalCallback {
		@Override
		@Nullable
		public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
					MethodProxy cglibMethodProxy) throws Throwable {

			ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
			String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

			// isScopedProxy 参考关于scope部分
			if (BeanAnnotationHelper.isScopedProxy(beanMethod)) {
				String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
				if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
					beanName = scopedBeanName;
				}
			}

            //若当前bean是工厂bean并且存在于bf中则返回一个代理对象
			if (factoryContainsBean(beanFactory, BeanFactory.FACTORY_BEAN_PREFIX + beanName) &&
					factoryContainsBean(beanFactory, beanName)) {
				Object factoryBean = beanFactory.getBean(BeanFactory.FACTORY_BEAN_PREFIX + beanName);
				if (factoryBean instanceof ScopedProxyFactoryBean) {
					// Scoped proxy factory beans are a special case and should not be further proxied
				}
				else {
					// It is a candidate FactoryBean - go ahead with enhancement
					return enhanceFactoryBean(factoryBean, beanMethod.getReturnType(), beanFactory, beanName);
				}
			}
            //若当前执行的函数对应的bean就是此次应该创建的bean,触发于context#refrsh,直接调用config中的本身函数逻辑
			if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
				// The factory is calling the bean method in order to instantiate and register the bean
				// (i.e. via a getBean() call) -> invoke the super implementation of the method to actually
				// create the bean instance.
				if (logger.isInfoEnabled() &&
						BeanFactoryPostProcessor.class.isAssignableFrom(beanMethod.getReturnType())) {
					logger.info(String.format("@Bean method %s.%s is non-static and returns an object " +
									"assignable to Spring's BeanFactoryPostProcessor interface. This will " +
									"result in a failure to process annotations such as @Autowired, " +
									"@Resource and @PostConstruct within the method's declaring " +
									"@Configuration class. Add the 'static' modifier to this method to avoid " +
									"these container lifecycle issues; see @Bean javadoc for complete details.",
							beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName()));
				}
				return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
			}
            //其他情况,直接通过config#method这样调用为了避免多次创建
			return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
		}
}    
```
大致逻辑如下:
```flow
st=>start: 开始
ed=>end: 结束
op0=>operation: 任意逻辑调用函数
con1=>condition: 是否refresh触发
op1=>operation: config中逻辑new创建
con2=>condition: 是否为手动调用
con3=>condition: 是否为factoryBean,并且于bf中存在
op2=>operation: 返回直接bf.getBean的代理逻辑工厂对象
op3=>operation: 嵌套调用,直接取bf.getBean
st(right)->op0->con1
con1(yes)->op1->ed
con1(no)->con2
con2(yes)->con3
con3(yes)->op2->ed
con3(no)->op3->ed
```
描述如下:
```java
@Bean
public Bean bean(){
  return new Bean();
}
@Bean
public Bean2() bean2(){
    return Bean2(bean())
}
```
1. 上述代码第一种非常简单,当`context#refresh`的时候会执行`bean()`函数,并且将对象放到bf中.
2. 当`bean2()`执行的过程,存在了嵌套`bean()`的调用,执行`bean()`的时候会导致`resolveBeanReference`的执行
    2.1 `[0]`逻辑执行的时候,如果该嵌套对象已经创建过则直接取缓存,否则会导致对应的工厂bd-->调用到1里边去,总之都会防止重复创建
```java
private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
				ConfigurableBeanFactory beanFactory, String beanName) {

			// The user (i.e. not the factory) is requesting this bean through a call to
			// the bean method, direct or indirect. The bean may have already been marked
			// as 'in creation' in certain autowiring scenarios; if so, temporarily set
			// the in-creation status to false in order to avoid an exception.
			boolean alreadyInCreation = beanFactory.isCurrentlyInCreation(beanName);
			try {
				if (alreadyInCreation) {
					beanFactory.setCurrentlyInCreation(beanName, false);
				}
				boolean useArgs = !ObjectUtils.isEmpty(beanMethodArgs);
				if (useArgs && beanFactory.isSingleton(beanName)) {
					// Stubbed null arguments just for reference purposes,
					// expecting them to be autowired for regular singleton references?
					// A safe assumption since @Bean singleton arguments cannot be optional...
					for (Object arg : beanMethodArgs) {
						if (arg == null) {
							useArgs = false;
							break;
						}
					}
				}
                //[0]: 从bf中尝试获取
				Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
						beanFactory.getBean(beanName));
				if (!ClassUtils.isAssignableValue(beanMethod.getReturnType(), beanInstance)) {
					// Detect package-protected NullBean instance through equals(null) check
					if (beanInstance.equals(null)) {
						if (logger.isDebugEnabled()) {
							logger.debug(String.format("@Bean method %s.%s called as bean reference " +
									"for type [%s] returned null bean; resolving to null value.",
									beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
									beanMethod.getReturnType().getName()));
						}
						beanInstance = null;
					}
					else {
						String msg = String.format("@Bean method %s.%s called as bean reference " +
								"for type [%s] but overridden by non-compatible bean instance of type [%s].",
								beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName(),
								beanMethod.getReturnType().getName(), beanInstance.getClass().getName());
						try {
							BeanDefinition beanDefinition = beanFactory.getMergedBeanDefinition(beanName);
							msg += " Overriding bean of same name declared in: " + beanDefinition.getResourceDescription();
						}
						catch (NoSuchBeanDefinitionException ex) {
							// Ignore - simply no detailed message then.
						}
						throw new IllegalStateException(msg);
					}
				}
				Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
				if (currentlyInvoked != null) {
					String outerBeanName = BeanAnnotationHelper.determineBeanNameFor(currentlyInvoked);
					beanFactory.registerDependentBean(beanName, outerBeanName);
				}
				return beanInstance;
			}
			finally {
				if (alreadyInCreation) {
					beanFactory.setCurrentlyInCreation(beanName, true);
				}
			}
		}
```
工厂对象的部分处理
```java
@Bean
public void FactoryBean<Test> test(){
    return new TestFactoryBean();
}
```
1. 若定义一个工厂bean,在refresh过程没有变化先通过new创建
2. 当嵌套调用或者主动调用会返回一个代理对象,并且该对象不会被放到bf缓存中,bf缓存
中放置一直都是第一步的工厂缓存,而这个代理对象的`getObject`会被处理成`bf.getBean`
### 总结
spring将配置类的每个`@Bean`通过cglib做了代理,防止了调用函数时重复创建的问题,并且需要注意该代理对象的接口并不是两层嵌套,不会出现内部调用的this失效问题

```flow
st=>start: Start|past:>http://www.google.com[blank]
e=>end: End|future:>http://www.google.com
op1=>operation: My Operation|past
op2=>operation: Stuff|current
sub1=>subroutine: My Subroutine|invalid
cond=>condition: Yes
or No?|xx:>http://www.google.com
c2=>condition: Good idea|rejected
io=>inputoutput: catch something...|future

st->op1(right)->cond
cond(yes, right)->c2
cond(no)->sub1(left)->op1
c2(yes)->io->e
c2(no)->op2->e
```