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
	sur.user_id ,
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
1. uuid不是顺序(递增或递减),导致生成的b树占用空间大,查询速度不高
2. uuid的长度过长