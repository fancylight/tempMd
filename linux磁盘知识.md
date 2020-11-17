## 知识点
### 磁盘及分区
#### 磁盘
linux下所有设备都被抽象成文件,保存在`/dev`目录下,`/dev/sd[a-p]`表示磁盘
#### 分区
MBR和GTP分区机制
- MBR分区
  - 磁盘=主分区+扩展分区 <=4
  - 拓展分区不能直接使用,需要继续添加逻辑分区才能使用,数据一般存在拓展分区中
  - 原理图 [参考](https://www.linuxprobe.com/chapter-06.html)
  ![图](https://www.linuxprobe.com/wp-content/uploads/2015/01/%E7%A1%AC%E7%9B%98%E7%9A%84%E6%89%87%E5%8C%BA.png)
- 分区
    - `fdisk /dev/sda`
    - n:创建新分页
    - 设置起始和结束
    - 写入
- 分区中创建文件系统
    - mkfs 类型 分区名称
    - ext3 ext4 ...
- 挂载
    - mount 分区 目录
    - 编辑/etc/fstab完成自动挂载    