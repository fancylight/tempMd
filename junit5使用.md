---
layout: post
title: junit5使用
date: 2020-12-01 09:39:25
tags: 测试
---
# 概述
用来记录关于junit5框架,junit5由三个部分构成,这三部分在maven中有不同的groupId标识,每个部分又存在不同的模块  
- `org.junit.platform`:测试运行平台,兼容junit4,支持在不同的ide以及不同的构建工具下运行
- `org.junit.jupiter`:定义了模型和拓展,运行在platfrom中
- `org.junit.vintage`:兼容junit4,junit3
## 基本概念
- `Test Class`:拥有至少一个测试函数的类,并且该类为一般类,静态成员类,或者`@Nested` 修饰的类
- `Test Method`:任何被`@Test`,`@RepeatedTest`,`@ParameterizedTest`,`@TestFactory`,`@TestTemplate`修饰的实例方法
- `Lifecycle Method`:任何被`BeforeAll`,`BeforeEach`,`AfterAll`,`AfterEach`修饰的函数
## 注解使用
### 基本注解 
`@Test`,`@BeforeAll`,`@AfterAll`,`@BeforeEach`,`@AfterEach`,用来在测试方法前后执行
`@DispalyName`:用来描述方法或者类在junit5中的名称,可以使用使用`@DisplayNameGeneration`来指定名称生成器
### @tag
该注解用来标记测试方法,用来分组,当执行的配置`idea`或者`maven`的`surefire`插件来决定使用那些
### @Disabled
该注解用来忽略哪些测试用例不执行
### 条件注解
- @EnabledOnOs:当执行操作系统时执行
- @EnabledForJreRange:指定jre范围执行
- @EnabledIf("methodName"):当执行的函数返回true时执行,若该注解定义在类上,则条件函数必须为static
### 顺序
参考[junit5顺序执行](https://junit.org/junit5/docs/current/user-guide/#writing-tests-test-execution-order)
### 生命周期
@TestInstance(TestInstance.Lifecycle.PER_CLASS):表示一个测试类仅仅创建一个实例
## 特性
### 过滤执行
配合`@Tag`注解来确定测试哪些
- 通过idea
{% asset_img 2020-12-01-09-48-27.png %}
该配置表示执行所有的`@Tag(versoin1.0)`的测试函数
- 通过maven插件
```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.0.0-M5</version>
                <executions>
                    <execution>
                        <id>default-test</id>
                        <phase>test</phase>
                        <goals>
                            <goal>test</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <groups>version1.0</groups>
                    <argLine>-Dfile.encoding=UTF-8</argLine>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
ps:其中`version`要使用2.2以上,`argLine`是用来处理测试输出乱码问题
### 断言机制
junit5中提供了`Assertion`实现了断言机制,并能够使用lambda表达式
- AssertAll
```java
@Test
    @DisplayName("分组测试")
    public void groupTest(){
        Assertions.assertAll("allTest",
                ()->Assertions.assertTrue(2<1,"2应该大于1") //该断言不成立,不会影响后续断言执行
                ,()->Assertions.assertTrue(()->{
                    System.out.println("子逻辑执行了");
                    return 1==1;
                },"1应该等于1"));
    }
```
### 假设机制
该机制`Assumptions`导致未通过的测试函数,会在总体结果中体现为ignore的状态,不会应该总的结果
```java
public class AssumeTest {
    @Test
    public void test(){
        Assumptions.assumeTrue(()->1>2,"1小于2");
    }
    @Test
    public void test2(){
        Assertions.assertEquals(1,1,"1等于1");
    }
}
```
idea运行结果  
{% asset_img 2020-12-16-15-36-23.png %}
如果是断言未通过
```java
public class AssumeTest {
    @Test
    public void test(){
        Assertions.assertEquals(1,2,"1等于1");
    }
    @Test
    public void test2(){
        Assertions.assertEquals(1,1,"1等于1");
    }
}

```
断言未通过
{% asset_img 2020-12-16-15-37-43.png %}