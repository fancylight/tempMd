# docker命令
- 提交
docker commit -a 作者 -m 说明 容器id 镜像名称
- 导出和导入镜像
	- 导出:docker export 镜像id > 路径/xx.tar
	- 导入:docker import xx.tar