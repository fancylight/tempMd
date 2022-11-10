## 概述
用来研究springMVC中关于请求匹配的一些问题
### 请求匹配逻辑
#### 请求匹配的逻辑
- `AbstractHandlerMethodMapping<T>`:该类是mvc关于方法匹配的核心类,其中`MappingRegistry`存放了所有控制层映射
- `RequestMappingInfo`:是mvc中一种重要的`Mapping`,当不同情况的请求进入mvc,如何判断url和最终执行函数的匹配是十分关键的
```java
public class AbstractHandlerMethodMapping<T> 
{
class MappingRegistry {
        //映射对象
		private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
        //直接路径匹配
		private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();
        //名称匹配
		private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();
        //跨域
		private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

		private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
}  

	static class MappingRegistration<T> {
        //子类mapping描述,一般就是指    RequestMappingInfo
		private final T mapping;
        //对于用户函数的抽象
		private final HandlerMethod handlerMethod;

		private final Set<String> directPaths;
    }  
    //匹配的请求描述
    private class Match {

		private final T mapping;

		private final MappingRegistration<T> registration;
	}   
}
    
```
- 匹配过程
```java
//AbstractHandlerMethodMapping-->从DispathcerServlet过来
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<>();
        //[1]能否直接路径匹配,即mapping没有使用占位符
		List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
		if (directPathMatches != null) {
			addMatchingMappings(directPathMatches, matches, request);
		}
        //[2] 如果直接匹配失败,则将所有的Mapping拿去匹配
		if (matches.isEmpty()) {
			addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
		}
        //[3] 
		if (!matches.isEmpty()) {
			Match bestMatch = matches.get(0);
			if (matches.size() > 1) {
				Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
				matches.sort(comparator);
				bestMatch = matches.get(0);
				if (logger.isTraceEnabled()) {
					logger.trace(matches.size() + " matching mappings: " + matches);
				}
				if (CorsUtils.isPreFlightRequest(request)) {
					for (Match match : matches) {
						if (match.hasCorsConfig()) {
							return PREFLIGHT_AMBIGUOUS_MATCH;
						}
					}
				}
				else {
					Match secondBestMatch = matches.get(1);
					if (comparator.compare(bestMatch, secondBestMatch) == 0) {
						Method m1 = bestMatch.getHandlerMethod().getMethod();
						Method m2 = secondBestMatch.getHandlerMethod().getMethod();
						String uri = request.getRequestURI();
						throw new IllegalStateException(
								"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
					}
				}
			}
			request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());
			handleMatch(bestMatch.mapping, lookupPath, request);
			return bestMatch.getHandlerMethod();
		}
        //还是没有匹配结果,返回没有匹配的原因(抛出对于异常)
		else {
			return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
		}
	}
        //[4] 进一步便利所有的mappings,让观察哪一个可以满足匹配
    	private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
		for (T mapping : mappings) {
			T match = getMatchingMapping(mapping, request);
			if (match != null) {
				matches.add(new Match(match, this.mappingRegistry.getRegistrations().get(mapping)));
			}
		}
	}
```
说明:
1. `[1]`中为何出现了直接匹配却出现集合,那是因此http请求可以url一致,但是method不同
- Info中的匹配逻辑
```java
//按照不同部分进行匹配,全部成立等于匹配正确 
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
        //判断 请求和mapping的 http请求是否一致
		RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
		if (methods == null) {
			return null;
		}
		ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
		if (params == null) {
			return null;
		}
		HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
		if (headers == null) {
			return null;
		}
		ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
		if (consumes == null) {
			return null;
		}
		ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
		if (produces == null) {
			return null;
		}
		PathPatternsRequestCondition pathPatterns = null;
		if (this.pathPatternsCondition != null) {
			pathPatterns = this.pathPatternsCondition.getMatchingCondition(request);
			if (pathPatterns == null) {
				return null;
			}
		}
		PatternsRequestCondition patterns = null;
		if (this.patternsCondition != null) {
			patterns = this.patternsCondition.getMatchingCondition(request);
			if (patterns == null) {
				return null;
			}
		}
		RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
		if (custom == null) {
			return null;
		}
		return new RequestMappingInfo(this.name, pathPatterns, patterns,
				methods, params, headers, consumes, produces, custom, this.options);
	}

```
#### spring对于请求匹配的抽象定义
{% asset_img mvc补充篇-路径匹配/2022-08-25-14-58-41.png %}
`RequestCondition`:该类用来定义了对于http请求的请求,`Method`,`Header`等条件的匹配
### mvc5中`PathPattern`体系
1. `PathContainer`:描述前端请求的路径类
2. `PathElement`:实际用来匹配路径的表达式,内部使用链表结构
3. `PathPattern`: 包含`PathElement`,调用匹配逻辑

#### 补充以前不知道的
1. `RequestMapping.params`:实际开发意义不大
    1.1 定义`RequestMapping.params = {"env=dev","path=test"}`
    1.2 当且仅当前端传递 `url?env=不为dev&path=test`才能匹配该接口
