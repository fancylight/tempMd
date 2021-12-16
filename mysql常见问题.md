# mysql积累
## 函数
## 规范
- only_full_group_by
5.7以上mysql默认开启,规则如下:
1. order by后面的列必须是在select后面存在的
2. select、having或order by后面存在的非聚合列必须全部在group by中存在
注意:
当关联查询时,该规则仅仅对主表生效
```sql
--- 该语句能够执行,因为主表是sys_user_role
SELECT
	sur.user_id,
	sur.role_id,
	su.`name`
FROM
	sys_user_role sur
	LEFT JOIN sys_role sr ON sr.id = sur.role_id
	LEFT JOIN sys_user su ON su.id = sur.user_id 
WHERE
	sr.type = 0 
	GROUP BY sur.user_id,sur.role_id;
```	
### 主键问题
#### uuid不作为主键的原因
uuid不是顺序(递增或递减),导致生成的b树占用空间大,
#### 雪花算法
简而言之雪花算法本身的定义是64位数字:
![图片](https://img2020.cnblogs.com/blog/813155/202005/813155-20200511162334239-459232117.png)
---
### 模糊查询
`like %%`会导致产生全文检索
1. 可以使用full text索引

## 学习
编码问题:
1. mysql中utf-8等于utf-8mb3(即1-3字符),也就是说4字符编码的如emoji文字需要使用utf-8mb4编码
2. ASCII编码并不是所有编码的子集,如UTF-16使用2个字符为最小单位,并没有直接映射到,但是大部分编码规则都是兼容ASCII的
### innnoDB
行格式:将记录存放在磁盘中的存放方式称为行格式,如`Compact`,`Redundant`,`Dynamic`,`Compressed`
```sql
CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
ALTER TABLE 表名 ROW_FORMAT=行格式名称
```
页:INNODB采取16KB为一页在磁盘和内存之间进行数据交换
#### compact结构
可变列长度  null值列表 记录头  数据  
前两个部分分别是描述数据列中可变类型实际所占字节数,以及数据列中可为null的列,其中记录头较为复杂,为固定5字节的描述信息.  

- 三个特殊的列
	- DB_ROW_ID:当未定义主键,并且不存在unique则系统创建作为主键
	- DB_TRX_ID:事务id
	- DB_ROLL_PTR:回滚指针

定长和变长字段,例如`varchar(xx)`就是变长,xx表示0-xx个长度,而`Char(xx)`,表示该列就是要占用xx字符,当字符集是utf-8时,xx个utf-8字符的长度是不确定的(10-30),但是asc编码就是确定的(10个字节),因此若`Char`字段使用utf-8这种变长编码,那么在`可变列长度`就会记录
#### 行溢出数据
`varchar(M)`表示最多存放M个字符,但是varchar有着最大65535个字节的限制,因此在不同的字符集下,能够存在的字符数量不同,也就是M的最大值不同,但是由于65553包含了`数据部分`,`变长部分`,以及`null部分`,因此即使使用ascii编码M也小于65535
1. 由于mysql采用页交换(16KB即16384字节),如果发生跨页(65536>16384),则会采用数据列部分记录部分数据,再记录剩余数据偏移地址的方式
