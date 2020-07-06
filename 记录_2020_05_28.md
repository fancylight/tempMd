## xml概念
spring的配置文件为例子
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:task="http://www.springframework.org/schema/task"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd    
                        http://www.springframework.org/schema/context    
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd    
                        http://www.springframework.org/schema/mvc    
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
                        http://www.springframework.org/schema/task  
						http://www.springframework.org/schema/task/spring-task-3.2.xsd ">
						
</beans>						
```
参考[Schema讲解](https://www.cnblogs.com/DreamDrive/p/4184375.html)
- xmlns:命名空间,描述元素定义,一般其url指定的位置存在xsd资源
- Schema
	- Schema是通过xsd来描述xml的文件,比dtd文件要更新
	- xsi:schemaLocation url和xsd url相对应来表示xsd位置
	