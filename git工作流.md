## 不同的工作方式
### [gitFlow](https://www.cnblogs.com/jeffery-zou/p/10280167.html)
1. `master`: 接收`release`分支打标签;接收`hotfix`分支
2. `dev`: 由`master`创建;接收`feature`;接收`hotfix`;创建`realase`;创建`feature`;
3. `feature`: 由`dev`创建;合并到`dev`
4. `release`: 由`dev`创建;合并到`master`和`dev`