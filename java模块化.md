# jdk9模块化概念学习
## 简单使用
- 目录
    ```
    project
        |_____lib
        |_____mod
        |_____src
            |____com
            |    |_____light
            |          |____Test.java
            |____module-info.java
    ```
模块代码如下:
```java
    module com.test{
    
    }
```
Test.java如下:
```java
package com.light;
public class Test{
  public static void main(String...args){
     Module mod = Test.class.getModule();
     System.out.println(mod);
  }
}

```

模块名称命名规则和package相同,并不是说模块名要和包名一致,如jdk中的java.base模块
- 编译及其结果
`javac -d mod -sourcepath src src/*.java src/com/light/*.java`
说明: -d表示编译结果路径,-sourcepath指定源码路径,但是还要挨个指明文件夹
    ```
    mod
        |____com
        |     |____light
        |          |_____Test.class
        |____module-info.class
    ```
- 打包
`jar --create --file test.jar --main-class com.light.Test -C mod .`
- 运行模块jar
`java --module-path test.jar --module com.test`
- 打包成模块文件
`jmod create --class-path test.jar test.jmod`
---test
## jmod文件的作用
`jlink --module-path test.jmod --add-modules java.base,com.test --output jre/`
通过改命令可以用来封装出一个jre,改目录结构如下:
```
jre
   |____bin 
   |____conf
   |____include
   |____legal
   |____lib
```
` jre/bin/java --module com.test`可以执行刚才的模块
## 模块化权限访问
  个人理解jdk9模块化降低了jar包粒度,通过描述文件清晰的表达了依赖关系,这一点和maven等构建工具是不会产生冲突的,高版本的maven-compile-plugin能够按照pom模型使用jdk9的编译命令,也就是说它把通过cp来构建的逻辑改成了根据module来构建.

  同时由于module-info.java文件的引入,我觉得如果项目大量使用会导致开发复杂,整体来说模块化的好处和意义不能完全理解
