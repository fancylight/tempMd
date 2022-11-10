## 流程逻辑(基于netty)
1. `ReactorHttpHandlerAdapter`:netty接受http协议并调用该适配器,将netty请求转换成spring http
```java
public class ReactorHttpHandlerAdapter implements BiFunction<HttpServerRequest, HttpServerResponse, Mono<Void>> {
	private final HttpHandler httpHandler;

	@Override
	public Mono<Void> apply(HttpServerRequest reactorRequest, HttpServerResponse reactorResponse) {
		NettyDataBufferFactory bufferFactory = new NettyDataBufferFactory(reactorResponse.alloc());
		try {
			ReactorServerHttpRequest request = new ReactorServerHttpRequest(reactorRequest, bufferFactory);
			ServerHttpResponse response = new ReactorServerHttpResponse(reactorResponse, bufferFactory);

			if (request.getMethod() == HttpMethod.HEAD) {
				response = new HttpHeadResponseDecorator(response);
			}

			return this.httpHandler.handle(request, response)
					.doOnError(ex -> logger.trace(request.getLogPrefix() + "Failed to complete: " + ex.getMessage()))
					.doOnSuccess(aVoid -> logger.trace(request.getLogPrefix() + "Handling completed"));
		}
		catch (URISyntaxException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to get request URI: " + ex.getMessage());
			}
			reactorResponse.status(HttpResponseStatus.BAD_REQUEST);
			return Mono.empty();
		}
	}
}    
```
2. `HttpHandler`(定义在web模块中):接受处理之后的请求做一部分通用处理,将实际处理转交给`WebHandler`
    `HttpWebHandlerAdapter`: 连接`HttpHandler`和`WebHandler`
3. `WebHandler`:抽象层定义在web模块,不同的时候在不同模块,如webflux
    `DispatcherHandler`: 分发请求的地方
	`FilteringWebHandler`: 包含了`DefaultWebFilterChain`
4. `DefaultWebFilterChain`: 包含了`WebFilter`和`WebHandler`
逻辑为
```
HttpWebHandlerAdapter-->FilteringWebHandler -->链式调用内部得DefaultWebFilterChain-->触发Filter->最终调用DispatcherHandler
```	
-----
### gateway的逻辑
`RoutePredicateHandlerMapping`: gateway内部封装了`FilteringWebHandler`
```java
//此处来判断逻辑
	protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		// don't handle requests on management port if set and different than server port
		if (this.managementPortType == DIFFERENT && this.managementPort != null
				&& exchange.getRequest().getURI().getPort() == this.managementPort) {
			return Mono.empty();
		}
		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug(
								"Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}

					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for ["
								+ getExchangeDesc(exchange) + "]");
					}
				})));
	}
	protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		
		return this.routeLocator.getRoutes() ////获取所有的路由
				// individually filter routes so that filterWhen error delaying is not a
				// problem
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
					return r.getPredicate().apply(exchange);
				})
						// instead of immediately stopping main flux due to error, log and
						// swallow it
						.doOnError(e -> logger.error(
								"Error applying predicate for route: " + route.getId(),
								e))
						.onErrorResume(e -> Mono.empty()))
				// .defaultIfEmpty() put a static Route not found
				// or .switchIfEmpty()
				// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
				.next()
				// TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});
```
`FilteringWebHandler`: `WebHandler`的子类,然后不参与外逻辑,而位于`handlerMapping`中
```java
public class FilteringWebHandler implements WebHandler {
private final List<GatewayFilter> globalFilters;
//适配器,实际会调用内部的GlobalFilter
private static class GatewayFilterAdapter implements GatewayFilter {

		private final GlobalFilter delegate;

		GatewayFilterAdapter(GlobalFilter delegate) {
			this.delegate = delegate;
		}

		@Override
		public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
			return this.delegate.filter(exchange, chain);
		}

		@Override
		public String toString() {
			final StringBuilder sb = new StringBuilder("GatewayFilterAdapter{");
			sb.append("delegate=").append(delegate);
			sb.append('}');
			return sb.toString();
		}

	}
//责任链逻辑
private static class DefaultGatewayFilterChain implements GatewayFilterChain {

		private final int index;

		private final List<GatewayFilter> filters;

		DefaultGatewayFilterChain(List<GatewayFilter> filters) {
			this.filters = filters;
			this.index = 0;
		}

		private DefaultGatewayFilterChain(DefaultGatewayFilterChain parent, int index) {
			this.filters = parent.getFilters();
			this.index = index;
		}

		public List<GatewayFilter> getFilters() {
			return filters;
		}

		@Override
		public Mono<Void> filter(ServerWebExchange exchange) {
			return Mono.defer(() -> {
				if (this.index < filters.size()) {
					GatewayFilter filter = filters.get(this.index);
					DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this,
							this.index + 1);
					return filter.filter(exchange, chain);
				}
				else {
					return Mono.empty(); // complete
				}
			});
		}	
}
```
- `GatewayFilter`和`GlobalFilter`: 前者通过`GatewayFilterAdapter`适配器的形式调用`GlobalFilter`,`GatewayFilter`实际是外层通过责任链的方式调用的
### gateway组件
#### 路由Router
```java
public class Route implements Ordered {
	private final String id;

	private final URI uri;

	private final int order;
	//断言
	private final AsyncPredicate<ServerWebExchange> predicate;

	private final List<GatewayFilter> gatewayFilters;

	private final Map<String, Object> metadata;
}	
```
1. 定义: 请求匹配逻辑
```java
//RoutePredicateHandlerMapping
	protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		return this.routeLocator.getRoutes()
				// individually filter routes so that filterWhen error delaying is not a
				// problem
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
					//Router内部断言进行匹配
					return r.getPredicate().apply(exchange);
				})
						.doOnError(e -> logger.error(
								"Error applying predicate for route: " + route.getId(),
								e))
						.onErrorResume(e -> Mono.empty()))
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});

	}
```
2. 断言:
```java
//RouteDefinitionRouteLocator
	private AsyncPredicate<ServerWebExchange> lookup(RouteDefinition route, PredicateDefinition predicate) {
		//根据名称获取不同的断言工厂
		RoutePredicateFactory<Object> factory = this.predicates.get(predicate.getName());
		if (factory == null) {
			throw new IllegalArgumentException("Unable to find RoutePredicateFactory with name " + predicate.getName());
		}
		if (logger.isDebugEnabled()) {
			logger.debug("RouteDefinition " + route.getId() + " applying " + predicate.getArgs() + " to "
					+ predicate.getName());
		}

		// @formatter:off
		//此处代码非常有意思
		Object config = this.configurationService.with(factory) //构造泛型builder
				//属性注入
				.name(predicate.getName())
				.properties(predicate.getArgs())
				.eventFunction((bound, properties) -> new PredicateArgsEvent(
						RouteDefinitionRouteLocator.this, route.getId(), properties))
				//创建		
				.bind();
		// @formatter:on
		//将PredicateDefinition 转换成 lambda
		return factory.applyAsync(config);
	}
```
- `configurationService`的实现
	1. 由于不同的factory内部的配置不同,但是yaml文件是通过map来配置的,因此为了将map转换成不同的config写了一套抽样builder
	2. 灵活使用了泛型模板和`bindable`转换
- 对于gateway相关的配置
springboot在spring的基础上实现了更加复杂的 属性转换机制,待整理
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: test
          uri: http://localhost:8081
          predicates:
            - Path=/order-server/**

```	
对应的properties是`gatewayProperties`
```java
@ConfigurationProperties(GatewayProperties.PREFIX)
@Validated
public class GatewayProperties {
	private List<RouteDefinition> routes = new ArrayList<>();

	/**
	 * List of filter definitions that are applied to every route.
	 */
	private List<FilterDefinition> defaultFilters = new ArrayList<>();
}
public class RouteDefinition {
		private List<PredicateDefinition> predicates = new ArrayList<>();
}	
public class PredicateDefinition {
		@NotNull
	private String name;

	private Map<String, String> args = new LinkedHashMap<>();
		//提供了直接从 String-->转换过来的构造器,因此他的yaml可以写的比较特别
		public PredicateDefinition(String text) {
		int eqIdx = text.indexOf('=');
		if (eqIdx <= 0) {
			throw new ValidationException(
					"Unable to parse PredicateDefinition text '" + text + "'" + ", must be of the form name=value");
		}
		setName(text.substring(0, eqIdx));

		String[] args = tokenizeToStringArray(text.substring(eqIdx + 1), ",");

		for (int i = 0; i < args.length; i++) {
			this.args.put(NameUtils.generateName(i), args[i]);
		}
	}
}	
```
3. 路由和断言都是通过分别定义 `RouteDefinition`和`PredicateDefinition`,`gateway`启动,或者refresh刷新生产对应的`Router`和其内部的`Predicate`
	3.1 `RoutePredicateFactory`配合`ConfigurationService`来创建断言
	3.3 同一个路由可以支持多个断言
	3.2 关于内置断言得使用可以[参考](https://learnku.com/articles/70591)	
4. `GatewayFilterFactory`: 实现了`GatewayFilter`层级得过滤器,直接附带在`Route`上,内部也是lambda,可以对比上文提到得`GatewayFilterAdapter`
因此可以得知,spring gateway是以`Route`为单位,请求进行`断言`匹配,获取 全局定义的`GlobalFilter`转换成`GatewayFilterAdapter`以及定义在对应`Route`身上的`GatewayFilter`,然后按照责任链的方式执行
`GlobalFilter`:不需要定义在`Route`上,会直接作用于所有的`Route`
`GatewayFilter`:需要定义在`Route`上,是通过`GatewayFilterFactory`创建出来的
无论是webflux中的`WebFilter`还是上述两个Filter都是责任链,并且返回值都不是bool,因此完全是处理和增强逻辑,并没有拦截的逻辑
#### 动态路由相关
1. 路由定位器
![20221101150623](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5Cb80a54ea43005b11ff2ffa185325dccc.png)
外部采用缓存,当缓存失效则会调用内部的`RouteLocator`调用内部的`RouteDefinitionRouteLocator`来获取`RouteDefinition`从而创建`Route`
2. 动态路由
`RouteDefinitionRouteLocator`:结构如下,这些组件会从不同的位置读取或者创建`RouteDefinition`,动态的关键在于如何能够感知到定义的变化
![20221101150948](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5C7a5a1557c114a2a1c8bd61756a6f749c.png)
- 基于库:`RedisRouteDefinitionRepository`基于配置的key名称读取redis数据的定义
- 基于配置中心:
	- `DiscoveryClientRouteDefinitionLocator`:读取配置中心服务名称来创建路由定义
```java
//CachingRouteLocator
public class CachingRouteLocator
		implements Ordered, RouteLocator, ApplicationListener<RefreshRoutesEvent>, ApplicationEventPublisherAware {
	private final RouteLocator delegate;

	private final Flux<Route> routes;

	private final Map<String, List> cache = new ConcurrentHashMap<>();

	public Flux<Route> getRoutes() {
		return this.routes;
	}
	//只有当发生了RefreshRoutesEvent才会导致缓存刷新
	public void onApplicationEvent(RefreshRoutesEvent event) {
		try {
			fetch().collect(Collectors.toList()).subscribe(
					list -> Flux.fromIterable(list).materialize().collect(Collectors.toList()).subscribe(signals -> {
						applicationEventPublisher.publishEvent(new RefreshRoutesResultEvent(this));
						cache.put(CACHE_KEY, signals);
					}, this::handleRefreshError), this::handleRefreshError);
		}
		catch (Throwable e) {
			handleRefreshError(e);
		}
	}		
}			
```	
- `RefreshRoutesEvent`发生条件
	- 容器启动(refresh)
	- 配置中心如nacos服务上下线|配置文件更改(服务器会向nacos客户端发出请求)
	```java
		//向nacos添加一个自定义监听器
	  public void dynamicRouteByNacosListener() {
        try {
            ConfigService configService = NacosFactory.createConfigService(serverAddr);
            configService.getConfig(dataId, group, 5000);
            configService.addListener(dataId, group, new Listener() {
                @Override
                public void receiveConfigInfo(String configInfo) {
                    clearRoute();
                    try {
                        List<RouteDefinition> gatewayRouteDefinitions = JSONObject.parseArray(configInfo, RouteDefinition.class);
                        for (RouteDefinition routeDefinition : gatewayRouteDefinitions) {
                            addRoute(routeDefinition);
                        }
			//为了保证对应缓存刷新,主动发起 RefreshRoutesEvent 事件
                        publish();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public Executor getExecutor() {
                    return null;
                }
            });
        } catch (NacosException e) {
            e.printStackTrace();
        }
    }
	```
#### gateway过滤器
由上文可以gateway的过滤器分为两类`GatewayFilter`和`GlobaleFilter`,虽然是两个接口实际上最终都会变成`GatewayFilter`,并且依附于`Route`上
`GlobaleFilter`:由自动配置类直接配置,或者用`defaultFilters`配置
![20221101153201](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5Ccde90ee15a4f357ad649c5a996c34fd7.png)
`GatewayFilter`:需要配置在对应的`Route`上,通过`GatewayFilterFactory`构建
![20221101153314](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5C218bbf4f6f9a9b73e5a41eca98d09edd.png)
gateway的各种能力都是通过过滤器实现的
如下为默认的全局过滤器,通过`Order`接口进行的排序:
![20221104153522](http://fancylight-markdown.oss-cn-hangzhou.aliyuncs.com/markdown%5C978a019f2c0f4b884fda7721930230c5.png)
- `NoLoadBalancerClientFilter`:是路由转发并没有在注册中心的加持下;
- `ReactiveLoadBalancerClientFilter`:通过注册中心获取实例ip
- `NettyRoutingFilter`:真正发送http请求的位置