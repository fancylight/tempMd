# 概述
该功能定义在`spring-cloud-context`中
## Refresh体系
1. 通过配置中心发送请求,对应服务内部发起`RefreshEvent`事件
2. `RefreshEventListener` 响应事件
3. `RefreshScope` 刷新 env,清除context中的scope缓存
### 特殊的RefreshScope Bean
1. 被注解`@RefreshScope`修改的`@Bean`函数首页会创建出`ScopedProxyFactoryBean`类型的bean
2. `GenericScope`会将上述的`BD`改造为`LockedScopedProxyFactoryBean`
```java
//GenericScope(BeanDefinitionRegistryPostProcessor 子类)
//子类RefreshScope
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		for (String name : registry.getBeanDefinitionNames()) {
			BeanDefinition definition = registry.getBeanDefinition(name);
			if (definition instanceof RootBeanDefinition) {
				RootBeanDefinition root = (RootBeanDefinition) definition;
                //bd是带有一层包装并且是ScopedProxyFactoryBean,被注解@RefreshScope修饰的
				if (root.getDecoratedDefinition() != null && root.hasBeanClass()
						&& root.getBeanClass() == ScopedProxyFactoryBean.class) {
					if (getName().equals(root.getDecoratedDefinition().getBeanDefinition().getScope())) {
                        //将bd中BeanClass重新设置
						root.setBeanClass(LockedScopedProxyFactoryBean.class);
                        //构造器参数设置一个当前的scope对象
						root.getConstructorArgumentValues().addGenericArgumentValue(this);
						// surprising that a scoped proxy bean definition is not already
						// marked as synthetic?
						root.setSynthetic(true);
					}
				}
			}
		}
	}

```
3. `LockedScopedProxyFactoryBean`逻辑
### 补充ScopedProxyFactoryBean 的创建逻辑
1. 复习springboot的bd的加载
 `ConfigurationClassPostProcessor`:是最初的BeanFactoryPost
 `AnnotationBeanDefinitionRegistryPostProcessor`:是后面创建出来的,会去调用ClassPathMapperScanner
2. 代理bd的创建逻辑
```java
//ClassPathMapperScanner 此处的触发逻辑是 BeanFactoryPost处理,不会去处理@Bean
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    //省略大部分
    //beanDefinitions 是通过扫码得到的,@Bean注解的函数会带有不同的scope,@RefreshScope实际是嵌套了@Scope的注解
     if (!definition.isSingleton()) { //单例的话则会通过ScopedProxyUtils 来处理
        BeanDefinitionHolder proxyHolder = ScopedProxyUtils.createScopedProxy(holder, registry, true);
        if (registry.containsBeanDefinition(proxyHolder.getBeanName())) {
          registry.removeBeanDefinition(proxyHolder.getBeanName());
        }
        registry.registerBeanDefinition(proxyHolder.getBeanName(), proxyHolder.getBeanDefinition());
      }
}
//ConfigurationClassBeanDefinitionReader
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
    // 创建代理对象
    		BeanDefinition beanDefToRegister = beanDef;
		if (proxyMode != ScopedProxyMode.NO) {
            //创建代理
			BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
					new BeanDefinitionHolder(beanDef, beanName), this.registry,
					proxyMode == ScopedProxyMode.TARGET_CLASS);
            //将原来的对象包装一层        
			beanDefToRegister = new ConfigurationClassBeanDefinition(
					(RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata, beanName);
		}

}
//ScopeProxyUtils
public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {
               	String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.
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
        //创建了一个 名称为 scopedTarget.name的新bean注册,beanClass 为 ScopedProxyFactoryBean
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases()); 
            }                    
```
3. 如果走`ConfigurationClassBeanDefinitionReader`会导致ioc中存在 `beanA`和`scopedTarget.beanA`两个bean,
并且`beanA`会包含`scopedTarget.beanA`;
其中 `beanA`的beanClass为原始对象;`scopedTarget.beanA` 则变成了`ScopedProxyFactoryBean`
如果执行了`GenericScope`的话,则会将 `beanA` 改成 beanClass `LockedScopedProxyFactoryBean`;
4. `ScopedProxyFactoryBean`的逻辑
```java
//ScopedProxyFactoryBean
	private String targetBeanName;

	/** The cached singleton proxy. */
	@Nullable
	private Object proxy;

    	public void setBeanFactory(BeanFactory beanFactory) {
		if (!(beanFactory instanceof ConfigurableBeanFactory)) {
			throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
		}
		ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;

		this.scopedTargetSource.setBeanFactory(beanFactory);

		ProxyFactory pf = new ProxyFactory();
		pf.copyFrom(this);
		pf.setTargetSource(this.scopedTargetSource);

		Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
		Class<?> beanType = beanFactory.getType(this.targetBeanName);
		if (beanType == null) {
			throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
					"': Target type could not be determined at the time of proxy creation.");
		}
		if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
			pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
		}

		// Add an introduction that implements only the methods on ScopedObject.
		ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
        //DelegatingIntroductionInterceptor的逻辑就是会执行ScopedObject的逻辑
		pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

		// Add the AopInfrastructureBean marker to indicate that the scoped proxy
		// itself is not subject to auto-proxying! Only its target bean is.
		pf.addInterface(AopInfrastructureBean.class);

		this.proxy = pf.getProxy(cbf.getBeanClassLoader());
	}

    //返回一个代理对象
	@Override
	public Object getObject() {
		if (this.proxy == null) {
			throw new FactoryBeanNotInitializedException();
		}
		return this.proxy;
	}
//DefaultScopedObject

    //可以直观的看到,会每次获取新的bean去执行逻辑
	@Override
	public Object getTargetObject() {
		return this.beanFactory.getBean(this.targetBeanName);
	}

	@Override
	public void removeFromScope() {
		this.beanFactory.destroyScopedBean(this.targetBeanName);
	}
```
5. `LockedScopedProxyFactoryBean`的逻辑 
```java
public static class LockedScopedProxyFactoryBean<S extends GenericScope> extends ScopedProxyFactoryBean
			implements MethodInterceptor {

		private final S scope;

		private String targetBeanName;
        // 持有之前的Scope对象,该对象是存在有 对应scopeBean的map对象的
		public LockedScopedProxyFactoryBean(S scope) {
			this.scope = scope;
		}

		@Override
		public void setBeanFactory(BeanFactory beanFactory) {
			super.setBeanFactory(beanFactory);
			Object proxy = getObject();
			if (proxy instanceof Advised) {
				Advised advised = (Advised) proxy;
				advised.addAdvice(0, this);
			}
		}

		@Override
		public void setTargetBeanName(String targetBeanName) {
			super.setTargetBeanName(targetBeanName);
			this.targetBeanName = targetBeanName;
		}
        //此中对象的代理逻辑
		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			Method method = invocation.getMethod();
			if (AopUtils.isEqualsMethod(method) || AopUtils.isToStringMethod(method)
					|| AopUtils.isHashCodeMethod(method) || isScopedObjectGetTargetObject(method)) {
				return invocation.proceed();
			}
			Object proxy = getObject();
			ReadWriteLock readWriteLock = this.scope.getLock(this.targetBeanName);
			if (readWriteLock == null) {
				if (logger.isDebugEnabled()) {
					logger.debug("For bean with name [" + this.targetBeanName
							+ "] there is no read write lock. Will create a new one to avoid NPE");
				}
				readWriteLock = new ReentrantReadWriteLock();
			}
			Lock lock = readWriteLock.readLock();
			lock.lock();
			try {
				if (proxy instanceof Advised) {
					Advised advised = (Advised) proxy;
					ReflectionUtils.makeAccessible(method);
					return ReflectionUtils.invokeMethod(method, advised.getTargetSource().getTarget(),
							invocation.getArguments());
				}
				return invocation.proceed();
			}
			// see gh-349. Throw the original exception rather than the
			// UndeclaredThrowableException
			catch (UndeclaredThrowableException e) {
				throw e.getUndeclaredThrowable();
			}
			finally {
				lock.unlock();
			}
		}

		private boolean isScopedObjectGetTargetObject(Method method) {
			return method.getDeclaringClass().equals(ScopedObject.class) && method.getName().equals("getTargetObject")
					&& method.getParameterTypes().length == 0;
		}

	}
```
结论就是:@RefreshScope修饰的@Bean最终会被`LockedScopedProxyFactoryBean`处理成代理对象,会在每次执行函数之前加锁,由
内部的源去获取刷新后的对象,因此会尝试每次创建bean,从实现了刷新逻辑(并且这里并不是更新自身,而是嵌套)
### 大致结论
1. `@RefreshScope`表示该`@Bean`或者外部的类会被创建为嵌套代理对象
2. 懒刷新机制,当调用到对应`Bean`的函数时会导致内部嵌套对象的重新创建并调用内部对象的函数