# 概述
neo4j是一个图数据库,通过其概念描述节点之前关系
## 概念
[参考](https://neo4j.com/docs/getting-started/current/get-started-with-neo4j/)
- 节点Node
节点为基本实体,可以拥有属性property,其归属Lables,以及和其他节点的关系
- 标签Labels
用来描述节点归属,节点可以拥有一个或者多个节点
- 关系Relationship
关系具有方向,单向,双向,或指向自身,并且关系可以含有属性
- 属性Properties
属性是键值对piar,并且具有类型,如`string`,`number`,`boolean`
- 命名规则:该数据库大小写敏感
	- Lablel:建议使用`:VehicleOwner`而不是`:vehice_owner`
	- Relationship type:建议`:OWNS_VEHICLE`
	- Property:建议`firstName`而不是`first_name`
## Cypher
- 节点表达
`(节点名:标签:标签... {属性名:属性,....})`
- 关系
`-[关系名:动作名 {属性名:属性....}]`
- 节点关联
```
(keanu:Person:Actor {name:  "Keanu Reeves"} )
-[role:ACTED_IN     {roles: ["Neo"] } ]->
(matrix:Movie       {title: "The Matrix"} )
```
上述出现的关系名,节点名可以理解为id,并不起什么作用
### CREATE
`CREATE (p:Student {name:'hape',age:18)-[r:ROLE_IN]->(c:Class {class_name:'kaiziban'}) `
与之相对的有merge,该操作会判断节点是否存在,若存在则创建,否则不创建
```
MERGE (keanu:Person { name: 'Keanu Reeves' })
ON CREATE SET keanu.created = timestamp() //表示不存在节点
ON MATCH SET keanu.lastSeen = timestamp() //表示存在
RETURN keanu.name, keanu.created, keanu.lastSeen
```