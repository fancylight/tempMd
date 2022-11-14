# 拓展语法
## 图表
需要渲染器支持对应的语法渲染机制
### [flow](https://flowchart.js.org/)
语法:
```
nodeName=>nodeType: nodeText[|flowstate][:>urlLink]
1. 首先定义
2. 连接关系
```
元素:
```
startID=>start: 开始框
inputID=>inputoutput: 输入输出框
operationID=>operation: 操作框
conditionID=>condition: 条件框
subroutineID=>subroutine: 子流程
endID=> end:结束框文字

startID->inputID->operationID->conditionID
conditionID(yes)->endID
conditionID(no)->subroutineID
```
结果如下:
```flow
st=>start: 开始
ed=>end: 结束
io=>inputoutput: 输入
op=>operation: 操作
cod=>condition: 条件
st->io->op->ed
```