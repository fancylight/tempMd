---
title: maven配置相关
date: 2020-07-22 09:00:21
tags: 
---
# 概述
本文用来描述maven中setting.xml和pom相关联的一些配置项
## mirror和repository关系
### repository的作用
简单来说,用来定义下载上传的位置,maven会先使用pom-->user.setting--->setting--->central
若设置私服则需要提供账号密码
```xml
  <profiles>
        <profile>
            <id>dev</id>
           <repositories>
               <repository>
                   <id>nexus</id>
                   <url>http://192.168.0.80:8081/repository/maven-public/</url>
                   <snapshots>
                       <enabled>true</enabled>
                   </snapshots>
                   <releases>
                       <enabled>true</enabled>
                   </releases>
               </repository>
           </repositories>
        </profile>
    </profiles>
    <activeProfiles>
        <activeProfile>dev</activeProfile>
    </activeProfiles>
```
 maven如何确定访问哪一个仓库:按照配置的顺序进行查找,也就是说当寻找任意jar包,如果在一个repo下找到就不会继续寻找下一个
### mirror的作用
简单来说mirror就是用来描述对任意repo的代理,当maven访问该repo会访问mirror的镜像地址
```xml
<mirror>

<id>alimaven</id>

<name>aliyun maven</name>

<url>http://maven.aliyun.com/nexus/content/groups/public/</url>

<mirrorOf>central</mirrorOf>

</mirror>
```
上述配置表示当访问`central`仓库时,会访问镜像地址
### server
当使用私服的时候,有需要配置该变量,如下
```xml
<server>
		<id>nexusMirror</id>
		<username>admin</username>
		<password>admin</password>
	</server>
```
注意,此时的id一定要和访问的repo匹配,但是如果使用mirror就得和mirror的id匹配,否则会出现401错误
### maven 如何处理`Repo`的
[参考](https://www.cnblogs.com/ctxsdhy/p/8482725.html)
1. 初始化所有的`Repository`对象,若没有配置默认会有一个`Center`
2. 获取所有的`Mirror`对象
3. 处理每一个`Repository`的`Mirror`
    3.1 优先判断 `mirror of` 是否直接等于`Repos`的id
    3.2 判断如`*`,`!`是否能够匹配
    3.3 给`Repo`设置`Mirror`
简单来说maven就是从`Repo`下载文件,`Mirror`只是当和`Repo`匹配时改写获取url路径
### `Repo`的使用顺序
[参考](https://swenfang.github.io/2018/06/03/Maven-Priority/)
1. `本地仓库`>`profile定义的仓库`>`一般的远程仓库`>`默认的Central仓库`
2. 假设profile定义了两个仓库,如何取呢
### maven-metadata.xml 文件
```xml
 <profile>
            <id>nexus-dep-profile</id>
            <repositories>
                <repository>
                    <id>rpl-repositories</id>
                    <name>rpl-repositories</name>
                    <layout>default</layout>
                    <url>http://192.168.110.29:8081/repository/rpl-repositories/</url>
                    <releases>
                    <!-- 表示是否能从该仓库下载正式版 -->
                        <enabled>true</enabled>  
                        <!-- 表示更新策略 -->
                        <updatePolicy>always</updatePolicy>
                    </releases>
                </repository>
                <repository>
                    <id>rpl-snapshot-repositories</id>
                    <name>rpl-snapshot-repositories</name>
                    <layout>default</layout>
                    <url>http://192.168.110.29:8081/repository/rpl-snapshot-repositories/</url>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>never</updatePolicy>
                    </snapshots>
                </repository>
            </repositories>
        </profile>
```
- 思考一下:正式版的更新逻辑是啥,不是应该不用更新吗
实际上如下
1. maven根据`updatePolicy`确认更新策略,进行策略比对
2. 比对的方式为:若version为`RELEASE`或`快照版本`,会下载远程的`maven-metadata.xml`,和本地的`maven-metadata.xml`比较,从而
确认最新版本是哪一个,去下载对应的版本,并且更新本地的`maven-metadata.xml`文件
3. 比较的逻辑:
    3.1 如果本地进行过install,则会产生`maven-metadata-local.xml`,在一般情况即,不开启`强制更新`则会直接比较远程和local的,取大值,若本地大则不下载,若远程大则下载
    3.2 若未进行install,则会一直比较使用并且使用远程最大的
[参考](https://www.jianshu.com/p/6c5e2b7b9408)
### 依赖问题
#### _remote.repositories
该文件描述从何仓库下载jar
```
#NOTE: This is a Maven Resolver internal implementation file, its format can be changed without prior notice.
#Thu Sep 17 15:20:03 CST 2020
maven-compiler-plugin-3.1.pom>nexusMirror=  
maven-compiler-plugin-3.1.pom>nexus=
maven-compiler-plugin-3.1.jar>nexusMirror=
maven-compiler-plugin-3.1.pom>maven-public=
maven-compiler-plugin-3.1.jar>nexus=
maven-compiler-plugin-3.1.jar>maven-public=
```
上述表示曾经从三个仓库下载过依赖,当改变repo配置时,maven则会主动下载文件,如果此次下载出错,则会出现
`xxx.pom.lastupdate`文件,该文件描述此次下载失败的原因,若存在该文件则无法构建成功
#### 仓库处理顺序
[参考文章](https://swenfang.github.io/2018/06/03/Maven-Priority/)
`dependency:purge-local-repository`:删除项目中依赖,并重新下载依赖
```xml
    <repositories>
        <repository>
            <!-- 第三方库 -->
            <id>nexus2</id>
            <name>Team Nexus Repository</name>
            <url>http://218.95.173.34:9000/nexus/repository/public/</url>
        </repository>
    </repositories>
    <dependencies>
      <dependency>
            <!-- 权限基础包 -->
            <!-- <groupId>com.wonders</groupId>
             <artifactId>auths</artifactId>
             <version>20180713</version>-->

            <groupId>com.wonders</groupId>
            <artifactId>auths-nm</artifactId>
            <version>20191225</version>
       </dependency>    
     </dependencies>  
```
查看该命令日志
```
//显示本地仓库情况
//依赖仓库使用顺序
Repositories (dependencies): [nexus (http://192.168.0.80:8081/repository/maven-public/, default, releases+snapshots), nexus2 (http://218.95.173.34:9000/nexus/repository/public/, default, releases+snapshots), nexusMirror (http://192.168.0.80:8081/repository/maven-public/, default, releases)] 
//插件仓库使用顺序
[DEBUG] Repositories (plugins)     : [nexusMirror (http://192.168.0.80:8081/repository/maven-public/, default, releases)]
//首先下载所有的pom文件
[DEBUG] Using transporter WagonTransporter with priority -1.0 for http://192.168.0.80:8081/repository/maven-public/
[DEBUG] Using connector BasicRepositoryConnector with priority 0.0 for http://192.168.0.80:8081/repository/maven-public/
Downloading from nexus: http://192.168.0.80:8081/repository/maven-public/com/wonders/auths-nm/20191225/auths-nm-20191225.pom
[DEBUG] Writing tracking file C:\Users\86150\.m2\repository\com\wonders\auths-nm\20191225\auths-nm-20191225.pom.lastUpdated
[DEBUG] Using transporter WagonTransporter with priority -1.0 for http://218.95.173.34:9000/nexus/repository/public/
[DEBUG] Using connector BasicRepositoryConnector with priority 0.0 for http://218.95.173.34:9000/nexus/repository/public/
Downloading from nexus2: http://218.95.173.34:9000/nexus/repository/public/com/wonders/auths-nm/20191225/auths-nm-20191225.pom
Downloaded from nexus2: http://218.95.173.34:9000/nexus/repository/public/com/wonders/auths-nm/20191225/auths-nm-20191225.pom (14 kB at 156 kB/s)
[DEBUG] Writing tracking file C:\Users\86150\.m2\repository\com\wonders\auths-nm\20191225\_remote.repositories
[DEBUG] Writing tracking file C:\Users\86150\.m2\repository\com\wonders\auths-nm\20191225\auths-nm-20191225.pom.lastUpdated
....
//当所有pom文件啊处理结束之后,才会开始jar包的下载
[INFO] Resolving artifact: com.wonders:auths-nm:jar:20191225
```
这部分是maven尝试下载依赖私有依赖,按照 `dev`中nexus ,pom中nexus2顺序进行查找,结果在pom中定义的nexus中匹配到依赖,则进行下载
- 处理顺序为
`本地仓库 > 私服 （profile）> 远程仓库（repository）和 镜像 （mirror） > 中央仓库 （central）`