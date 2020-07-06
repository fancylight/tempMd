# 概述
分析mybatis源码
## 配置相关
### setting文件
- properties:读取properties文件用来设置属性
```xml
<configuration>
    <properties resource="jdbc.properties">

    </properties>
</configuration>	
```
- typeAliases:表示用字符串表示java类型
```xml
<configuration>
<typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
  <typeAlias alias="Post" type="domain.blog.Post"/>
  <typeAlias alias="Section" type="domain.blog.Section"/>
  <typeAlias alias="Tag" type="domain.blog.Tag"/>
</configuration>	
```
- typeHandlers:用来转换jdbc结果集