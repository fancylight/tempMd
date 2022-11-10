## Dockerfile编写
### FROM
集成基础镜像
### COPY 
`cop 上下文目录 宿主目录`
### WORKDIR 
`WORKDIR dir`:影响下一个命令的工作路径
```
mkdir /opt/java
workdir /opt/java
copy ./test.jar ./
```
### Volume
用于共享image和本机目录,持久化作用

```
# 指定一个容器目录,是为了防止用户启动的时候忘记设置 -v
# 当启动时,将dir绑定到宿主机的运行下一个目录
volume dir
```
和运行时`-v`的命令不同,前者会在宿主机生成一个卷目录,后者需要指定宿主机和容器
### CMD命令

## docker命令
### volume
`docker volume ls`:查看所有卷名称
`docker volume inspect volumename`:查看指定卷位置
### run
`docker run -it`:容器会打开并且运行在可以交互界面, 默认会运行`/bin/bash`
`docker run -d`:后台运行
`docker exec -it 命令`:连接上运行中的容器
### 上下文context-dir
执行`docker build`的时候应该指定一个上下文环境,当`COPY src target`这个src实际就是从上下文路径发送到docker的,
因此需要指定,一般情况都是.即表达为dockerfile文件所在位置