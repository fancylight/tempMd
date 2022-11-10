# 概述
记录linux名称
一般来说linux系统基本上分两大类：cat /etc/issue查看linux系统版本
RedHat系列：Redhat、Centos、Fedora等
Debian系列：Debian、Ubuntu等
## 系统本身学习
### linux 目录结构
[参考说明](https://blog.51cto.com/u_15083991/2739111)
1. `etc`:都是配置文件,如apt的配置文件等
2. `usr`: 应该理解为`unix system resources`,linux系统资源目录,因为user应该是`home`目录
3. `var`:一般是日志目录
4. `dev`:系统设备目录
## 搜索相关
### grep 
强大的搜索工具,能使用正则,并且能够将行打印处理
### which
可以用来搜索命令位置
## 信息命令
### ps
查看进程进程状态
- `a`:查看所有用户进程
- `x`:查看非终端进程
- `u`:以用户为主的格式来显示程序状况
## 浏览显示
### less
分屏显示
### more
分屏显示,该命令仅能向前滚动,h查看内置命令
### awk
逐行读取,并且默认按照`空格`分割每行数据
`awk -F '=' '{print $2}' fileName`:逐行读取`filename`.并且每行按照`=`分割,并且打印都打印出第二个变量
## 执行和脚本命令
### nohup
后台运行脚本
例子:`nohup ./start-dishi.sh >output 2>&1 &`
解释:
```
1或者2>文件或者/dev/null等设备路径,1表示标准输出到指定路径,2表示错误输出到指定路径
> 等价于1>,即标准输出
2>&1表示2即错误输出重定向到1,那么2也就输出到和1相同的路径下,由于2输出没有缓存若写出1>xx 2>xx,则会导致1,2输出覆盖
最后的&表示即使终端退出程序依旧执行
```
### systemd和init
[参考](https://segmentfault.com/a/1190000039989489)
1. `init`:是一个守护程序
2. `service`:则是通过init.d目录启动程序
启动程序
## 软件安装命令
### 几种安装方式的区别和联系
#### RPM和YUM,及APT
rpm即`Red-hat Package Manager`:即红帽开发的基层安装工具,他不会按照包之间依赖,不会处理这些问题
yum即`Yellow dog Updater, Modified`:基于rpm开发的能够处理包之间的依赖关系,会去服务器分析所有rpm包的关系再进行下载安装
apt:debian系linux中的包安装工具
### apt
debian默认的包管理
### dpkg
debian 基础包管理,和rpm一样
- 卸载方式
`dpkg -l | grep docker` 可以搜索关于docker的安装包-->`apt autoremove 对应的包`
1. 设置apt源
## wls2
[参考](https://blog.csdn.net/qq_32114645/article/details/124548058)
1. 如何开启`systemd`
```cmd
sudo vim /etc/wsl.conf
```
添加
```
[boot]
command="/usr/libexec/wsl-systemd"
```
重启