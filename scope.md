# 概述
研究spring中`Scope`注解的原理和相关拓展
## spring体系(除了spring-cloud)
`Scope`分为`prototype`,`singleton`,以及web环境下的`request`,`session`
### 实例的获取逻辑
```java
//AbstractBeanFactory
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

                    //省略....
                	if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new ScopeNotActiveException(beanName, scopeName, ex);
					}
				}
			}
            }                
```
`prototype`和`singleton`的取值逻辑取值逻辑比较传统;其他scope类型则是通过`Scope`来处理,实际和Bf类似都是创建和缓存实际的区别
1. 当缓存命中,直接取缓存(后两者实际就是到`request`或者`session`中取数据)
2. 当缓存没有命中,则去创建,并防止到缓存中
### 如何才能每次正确调用 getBean逻辑
1. 最简单结构
```java
@Scope(value = "request")
@Componet
public class ScopeTestBean {
    public void test() {

    }
}
@Service
@Slf4j
public class ScopeService {
    @Resource
    private ScopeTestBean scopeTestBean;
    public void scopeBeanHash() {
        scopeTestBean.test();
        log.info(scopeTestBean.hashCode() + "");
    }
}
@RestController
@Slf4j
public class ScopeController {
    @Resource
    private ScopeService scopeService;

    @GetMapping("scopeBeanHash")
    public void scopeBeanHash() {
        log.info(scopeService.hashCode() + "");
        scopeService.scopeBeanHash();
    }
}
//最终结果会导致启动异常,如下:

	public static RequestAttributes currentRequestAttributes() throws IllegalStateException {
        //由于启动的时候并存在request存在hold中,因此构建bean报错
		RequestAttributes attributes = getRequestAttributes();
		if (attributes == null) {
			if (jsfPresent) {
				attributes = FacesRequestAttributesFactory.getFacesRequestAttributes();
			}
			if (attributes == null) {
				throw new IllegalStateException("No thread-bound request found: " +
						"Are you referring to request attributes outside of an actual web request, " +
						"or processing a request outside of the originally receiving thread? " +
						"If you are actually operating within a web request and still receive this message, " +
						"your code is probably running outside of DispatcherServlet: " +
						"In this case, use RequestContextListener or RequestContextFilter to expose the current request.");
			}
		}
		return attributes;
	}
```
2. 延迟bean加载
```java
@Scope(value = "request")
@Componet
public class ScopeTestBean {
    public void test() {

    }
}
@Service
@Slf4j
public class ScopeService {
    @Resource
    private ScopeTestBean scopeTestBean;
    public void scopeBeanHash() {
        scopeTestBean.test();
        log.info(scopeTestBean.hashCode() + "");
    }
}
@RestController
@Slf4j
@Lazy
public class ScopeController {
    @Resource
    private ScopeService scopeService;

    @GetMapping("scopeBeanHash")
    public void scopeBeanHash() {
        log.info(scopeService.hashCode() + "");
        scopeService.scopeBeanHash();
    }
}
```
很多bean不会在context#refrsh的过程就创建
```java
//DefaultListableBeanFactory-->会被ApplicationContext#refresh调用
public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) { //单例&&抽象&&非懒加载
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
}        
```
如此修改存在两个问题
- hash值没有变化,说明scope没有生效
- controller->service->bean  这样的引用关系,为了递进lazy需要将所有最终引用了bean的组件都比标记为`@Lazy`
    - 实际就是没有地方去触发 `getBean`逻辑,因此无论是用`prototype`还是主动延迟`@Lazy`都没有实际解决问题
### `@Scope`注解的代理逻辑
#### 代理对象的创建逻辑
```java
//ScopedProxyCreator
	public static BeanDefinitionHolder createScopedProxy(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry, boolean proxyTargetClass) {
        //进入此处会导致创建代理BD, @Bean和其他如@Compnet,@Configuration等类型的bean都会调用此处                    
		return ScopedProxyUtils.createScopedProxy(definitionHolder, registry, proxyTargetClass);
	}
//AnnotationConfigUtils 
    //非@Bean的逻辑
	static BeanDefinitionHolder applyScopedProxyMode(
			ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {

		ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
		if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
			return definition;
		}
        //调用代理
		boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
		return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
	}
//ConfigurationClassBeanDefinitionReader
    //@Bean的逻辑    
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		// Replace the original bean definition with the target one, if necessary
		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata, beanName);
		}
    }        
```    
代理的行为:
```java
//ScopedProxyCreator
	public static BeanDefinitionHolder createScopedProxy(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		return ScopedProxyUtils.createScopedProxy(definitionHolder, registry, proxyTargetClass);
	}
//ScopedProxyUtils
	public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.
        //[1] 创建一个新的BD,并且是一个嵌套的BD
		RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
		proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
		proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
		proxyDefinition.setSource(definition.getSource());
		proxyDefinition.setRole(targetDefinition.getRole());

		proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
		if (proxyTargetClass) {
			targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
		}
		else {
			proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
		}

		// Copy autowire settings from original bean definition.
		proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
		proxyDefinition.setPrimary(targetDefinition.isPrimary());
		if (targetDefinition instanceof AbstractBeanDefinition) {
			proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
		}

		// The target bean should be ignored in favor of the scoped proxy.
		targetDefinition.setAutowireCandidate(false);
		targetDefinition.setPrimary(false);

		// Register the target bean as separate bean in the factory.
        //[2] 将内部的嵌套对象注册
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).
        //[3] 返回新的代理 BeanClass为ScopedProxyFactoryBean的BD
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
	}
```
#### ScopedProxyFactoryBean代理对象
![20221109155725](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5Cb1b007d7fceb26a786b636c913eec8ee.png)
1. `FactoryBean`会首先创建放在Ioc容器中,并且会将`@beanName`覆盖成`beanName`
2. 当正确获取`Bean`时会通过`beanName`获取`FactoryBean`,最终导致`FactoryBean#getObject`调用返回`Bean`
```java
public class ScopedProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanFactoryAware, AopInfrastructureBean {
        //当该FactoryBean创建之后bf调用此处
        public void setBeanFactory(BeanFactory beanFactory) {
		if (!(beanFactory instanceof ConfigurableBeanFactory)) {
			throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
		}
		ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;

		this.scopedTargetSource.setBeanFactory(beanFactory);
        //主动调用ProxyFactory,详情可以参考aop分析
		ProxyFactory pf = new ProxyFactory();
		pf.copyFrom(this);
        //设置Proxy内部的target
		pf.setTargetSource(this.scopedTargetSource);

		Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
		Class<?> beanType = beanFactory.getType(this.targetBeanName);
		if (beanType == null) {
			throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
					"': Target type could not be determined at the time of proxy creation.");
		}
        //设置代理类的代理接口
		if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
			pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
		}

		// Add an introduction that implements only the methods on ScopedObject.
        //当代理接口为ScopedObject的时候会执行DelegatingIntroductionInterceptor逻辑,可以用来获取真正的proxy对象
		ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
		pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

		// Add the AopInfrastructureBean marker to indicate that the scoped proxy
		// itself is not subject to auto-proxying! Only its target bean is.
        //补充一个代理接口,这一个标记接口
		pf.addInterface(AopInfrastructureBean.class);

		this.proxy = pf.getProxy(cbf.getBeanClassLoader());
	}
        }        

//SimpleBeanTargetSource
public class SimpleBeanTargetSource extends AbstractBeanFactoryBasedTargetSource {
    //代理Proxy对象内部的target实际是通过ioc获取最新的
	@Override
	public Object getTarget() throws Exception {
		return getBeanFactory().getBean(getTargetBeanName());
	}
}    
//
public class DelegatingIntroductionInterceptor extends IntroductionInfoSupport
		implements IntroductionInterceptor {
        	private Object delegate;            
        	public Object invoke(MethodInvocation mi) throws Throwable {
        //只有当 delegate的接口函数调用的时候会执行代理逻辑                
		if (isMethodOnIntroducedInterface(mi)) {
			// Using the following method rather than direct reflection, we
			// get correct handling of InvocationTargetException
			// if the introduced method throws an exception.
			Object retVal = AopUtils.invokeJoinpointUsingReflection(this.delegate, mi.getMethod(), mi.getArguments());

			// Massage return value if possible: if the delegate returned itself,
			// we really want to return the proxy.
			if (retVal == this.delegate && mi instanceof ProxyMethodInvocation) {
				Object proxy = ((ProxyMethodInvocation) mi).getProxy();
				if (mi.getMethod().getReturnType().isInstance(proxy)) {
					retVal = proxy;
				}
			}
			return retVal;
		}

		return doProceed(mi);
	}
        }                    
```
### spring-cloud环境下的scope
1. 刷新环境,并且同事重新设置`@ConfigurationProperties`的类
    1.1 `RefreshEventListener`响应`RefreshEvent`,调用`RefreshScope`
    1.2  `RefreshScope` 
    ```java
    	public synchronized Set<String> refresh() {
        //这里会将所有 @ConfigurationProperties的配置类内容更新为最新的
		Set<String> keys = refreshEnvironment();
        //调用scope删除bean缓存
		this.scope.refreshAll();
		return keys;
	}
    ```
2. `@RefreshScope`对应的scope为`RefreshScope`,代理原理和上文没有区别
3. 特别情况:
由上文可以,`@Scope`的原理就是创建一个代理类,并且和`SimpleBeanTargetSource`配合从而实现动态更新
![20221109184152](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5C7bbb2bc6047e251df6e6e9fcf6135a52.png)
结果如上,当执行`beanName`的任意函数会导致代理取尝试获取名为`scopeTarget.beanName`的bean,然后去执行逻辑;
其中`beanName`对应的bd实际为`ScopedProxyFactoryBean`或者子类(`Cloud环境`)
其中`scopeTarget.beanName`对应的bd实际为`BeanClas`
```java
@Configuration
@Slf4j
@RefreshScope
public class AccessGatewayFilter implements GlobalFilter, Ordered {
}    
```  
如果如上代码这样定义会导致上图中的`scopeTarget.beanName`bd变成`ScopedProxyFactoryBean`或者子类(`Cloud环境`),结果会导致在`@Configuration`
内部
`ConfigurationClassEnhancer`此类导致的,改变了`@Configuration`bd中的class
### 总结
1. 原理就是通过代理,每个函数的执行之前会尝试获取新的内部target对象
2. 每次函数执行的过程实际是在target内部,所以只有target变动就可以连锁内部属性的变动