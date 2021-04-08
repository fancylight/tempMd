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
- 注册中心
- Feign
## jwt
- 关于token的原理
- jwt的使用
    - 这里写了一个mvc的Interceptor来校验用户是否登录
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