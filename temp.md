# 简单记录
## gateWay
- gateWay需要跨域支持么(没有使用ngix,因此需要)
- gateWay负载均衡
## plumelog-logback
分布式链路日志,数据(AppenderBase子类)写入到指定载体,该模块被大部分业务模块引用
## Database-Rider(数据库测试框架)
## 系统缺失功能
- 异常处理及公共异常
- 入参校验
- 统一用户获取
- dao未集成plus
## 基础功能
### 用户-角色-权限
- sys_usr <----sys_usr_role----->sys_usr_role
- sys_usr <----sys_usr_menu----->sys_menu
仅仅实现了菜单权限
### 字典表
dict
### 审核流程设定
`cost_audit_process`:按照type分组,step作为节点顺序
### 单元测试
- javaWeb开发是否需要分层进行公共函数测试
- 集成测试和单元测试的区别
-----
- dao层测试(h2,database-rider,flyway)