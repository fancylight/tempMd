---
layout: post
title: 记录shop-service项目
date: 2021-04-02 17:36:23
tags: 工作记录
---
# 概述
用来记录在阅读项目过程中没有使用过,或者不清楚的部分.
- 系统依赖关系:
```
                        mdj-common
                            |
            __________mdj-core-baisc
            |                 |
     mdj-core-invoicing  mdj-core-shop
            |               |
 application-invoicing  mdj-core-member______________ 
                            |                        |
                    mdj-shop-account     mdj-core-commmission
                            |                         |
                            |----------------- mdj-core-order      
                            |                         |
                     application-master      applicaiton-shop

```
## spring相关
### @Import注解的使用
- @Import注解的处理在`ConfigurationClassParser#processImports`函数中
- 该注解作用,会取该注解的value标注的类
    - 当类为ImportSelector,实际上就是第一个delegate,继续处理
    - 当类是ImportBeanDefinitionRegistrar,处理内部,注册更多的bd
    - 其他情况下,会将该类当作被标注了`@Configuration`
- 结合上述情况三,在项目内可以实现一个`@Enable`注解
```java
@Import(OperationConfig.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EnableOperationLog {
}
//配置类
//此处不需要写@Configuration注解
public class OperationConfig {
    @Bean
    public SimpleJdbc simpleJdbc(){
        return new SimpleJdbc();
    }
    @Bean
    public OperationLog operationLog(){
        OperationLog operationLog = new OperationLog();
        return operationLog;
    }
    @Bean
    public OperationAspect operationAspect(){
        return new OperationAspect();
    }
    @Bean
    public MybatisPlusInterceptor addInnerLogInterceptor(MybatisPlusInterceptor mybatisPlusInterceptor, OperationLog operationLog, SimpleJdbc simpleJdbc) {
        InnerOpLogInterceptor innerOpLogInterceptor = new InnerOpLogInterceptor();
        innerOpLogInterceptor.setOperationLog(operationLog);
        innerOpLogInterceptor.setSimpleJdbc(simpleJdbc);
        mybatisPlusInterceptor.addInnerInterceptor(innerOpLogInterceptor);
        return mybatisPlusInterceptor;
    }
}
```    
- @Bean注解修饰的方法,返回值对象,可以在其内部使用`@Autowired`,`@Value`等注解,原因是这两个注解生效的时期是ioc对象构造之后,通过Postprocessor处理的
### aop
- 切面表达式复习
## spring-cloud相关
- 注册中心 Netflix Eureka:用来将不同的微服务客户端注册到服务中心,服务中心本身也可以是服务端
- 负载均衡  Ribbon:存在多个相同微服务时,可以进行负载均衡访问
- 访问代理 Feign: 客户端调用通过注解避免Rest请求代码
- 熔断器 Hystrix:
- Zuul 网关
## jwt
- 关于token的原理
- jwt的使用
    - 这里写了一个mvc的Interceptor来校验用户是否登录
## 跨域问题    
- 参考阮一峰的文章,做个测试,简单请求和复杂请求
## swagger3
[使用参考](https://www.cnblogs.com/geekdc/p/13879913.html)    
主要是这里的访问路径和2不同了
## mybatis
- 实现动态刷新sql文件
- xml语法学习
- mybatis拦截器实现,以及如何获取最终的sql(拦截器中获取的sql是带有占位符的)
[获取最终sql](https://blog.csdn.net/w254826019/article/details/109745097)
- mybatis内部源码阅读
## http协议
### 头部
- x-forwarded-for
- Proxy-Client-IP
- WL-Proxy-Client-IP
## Validator
这里使用了自定义实现,并没有使用注解,但是他这个实现并没有完成.
## redis
### redis的key设计原则
## 杂项
### 设计
- 使用DTO作为入参和整体传递的实体(验证是否是正确的设计)
### fastjson
- 如何使用fastjson该api
## 权限相关
### 系统实现概述
- 顺序: `WebInterceptor`--->`PermissionAspect`校验
- 拦截器完成 名单验证,匿名验证,token校验,如果是通过token校验,则会将设置系统用户缓存,以及刷新redis缓存时间
- 切面:当通过拦截器后,若controller存在对应注解,则进一步验证该用户的权限能否执行该rest接口.
#### 注解
- 注解`@Anonymous`:表示匿名,不校验,在`PermissionAspect`切面和`WebInterceptor`拦截器都有校验
- 注解`@DomainPermission`:检测当前登录的User.isDomain(),Domain就是连锁店
- 注解`@ShopPermission`:判断当前操作的权限和缓存中的是否一致
## 前端部分
- vue的原理
- webpack相关
- vue项目结构