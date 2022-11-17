## spirngMVC相关
### 原生mvc
1. springMVC 触发配置的逻辑`WebMvcConfigurationSupport`
该类定义了mvc内部组件的创建逻辑,并且调用了抽象函数来自定义组件内部
```java
public class WebMvcConfigurationSupport{
    private List<HttpMessageConverter<?>> messageConverters;
    //子类实现
    	protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
	}
        //获取转换器
    	protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
    //定义RequestMappingHandlerAdapter
    	@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcValidator") Validator validator) {

		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
        //此处会触发获取转换器
		adapter.setMessageConverters(getMessageConverters());
		return adapter;
	}
    	@Bean
	public RouterFunctionMapping routerFunctionMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		RouterFunctionMapping mapping = new RouterFunctionMapping();
        //获取转换器
		mapping.setMessageConverters(getMessageConverters());
		return mapping;
	}
}
```
2.定义代理类
```java
//开启会将父类中的@Bean作用于容器中
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
        //将容器中的WebMvcConfigurer 放置到WebMvcConfigurerComposite(代理逻辑)
	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}        
}
```
3. 用户自定义接口
```java
public interface WebMvcConfigurer {
    //定了大量这样的接口
    	default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
	}
}
```
从1从可以看出来`@Bean`每个创建组件的逻辑中会导致用户自定义的代理多次调用,并且对应的用户定义的部分并不会进去到容器中.
### springBoot的拓展
定义在`WebMvcAutoConfiguration`中
1. 对于`WebMvcConfigurer`的增强,原意中并没有通过ioc扫描直接获取自定义组件并且放置到对于mvc内部组件,springboot做了部分拓展
```java
    //该类也变成了配置类
	@Configuration(proxyBeanMethods = false)
    //EnableWebMvcConfiguration 拓展了 DelegatingWebMvcConfiguration
	@Import(EnableWebMvcConfiguration.class)
	@EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
	@Order(0)
	public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
        //大量使用了 ObjectProvider,因此会扫描到容器中部分自定义组件
    		public WebMvcAutoConfigurationAdapter(WebProperties webProperties, WebMvcProperties mvcProperties,
				ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
				ObjectProvider<DispatcherServletPath> dispatcherServletPath,
				ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) {
			this.resourceProperties = webProperties.getResources();
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConvertersProvider = messageConvertersProvider;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
			this.dispatcherServletPath = dispatcherServletPath;
			this.servletRegistrations = servletRegistrations;
			this.mvcProperties.checkConfiguration();
		}
        //注意不是HttpMessageConverter
		private final ObjectProvider<HttpMessageConverters> messageConvertersProvider;
    	@Override
		public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
            //将容器中扫描到的转换器加入到list中
			this.messageConvertersProvider
					.ifAvailable((customConverters) -> converters.addAll(customConverters.getConverters()));
		}    
    }        
```
2. 拓展`DelegatingWebMvcConfiguration`
```java
	/**
	 * Configuration equivalent to {@code @EnableWebMvc}.
	 */
	@Configuration(proxyBeanMethods = false)
	@EnableConfigurationProperties(WebProperties.class)
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
    }
```
3. 由于springboot提供了一个`WebMvcAutoConfigurationAdapter` 这么一个`WebMvcConfigurer`,因此许多组件的顺序取决于`DelegatingWebMvcConfiguration`初始化顺序
### 常见的如`HttpMessageConverter`在springboot中问题
1. `HttpMessageConvertersAutoConfiguration`
springboot自动装配定义了部分的HttpMessageConverter
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HttpMessageConverter.class)
@Conditional(NotReactiveWebApplicationCondition.class)
@AutoConfigureAfter({ GsonAutoConfiguration.class, JacksonAutoConfiguration.class, JsonbAutoConfiguration.class })
//这些类提供了对于的HttpMessageConverter
@Import({ JacksonHttpMessageConvertersConfiguration.class, GsonHttpMessageConvertersConfiguration.class,
		JsonbHttpMessageConvertersConfiguration.class })
public class HttpMessageConvertersAutoConfiguration {
    //根据容器中HttpMessageConverter创建HttpMessageConverters
	@Bean
	@ConditionalOnMissingBean
	public HttpMessageConverters messageConverters(ObjectProvider<HttpMessageConverter<?>> converters) {
		return new HttpMessageConverters(converters.orderedStream().collect(Collectors.toList()));
	}
}    
```
- 上文定义代理类部分的循环非常重要,最终会决定内部 `HttpMessageConverter`的顺序
    - `DelegatingWebMvcConfiguration`初始化时决定`WebMvcConfigurer`的顺序
    - springboot中`WebMvcAutoConfigurationAdapter`的`order`为0,因此`HttpMessageConverters`中的`HttpMessageConverter`即直接配置于容器中的转换器顺序很高
    - 需要注意`HttpMessageConverters`实际上是被很多组件引用的,这部分的转换器实际存在于ioc容器中,而一般通过`WebMvcConfigurer`则是new出来的不会存在于容器中