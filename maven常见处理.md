# 描述常见的maven处理
## 常见问题
### maven依赖传递以及本地依赖问题
- `parent`和`module`
    - 前者表示继承父pom的依赖,插件等;后者表示构建父模块时会进行子模块的构建
- 依赖(本地)传递问题    
项目如下:
    ```
    common(父模块)
        |___lib(父模块本地依赖)
        |___A模块(子模块)
        |___B模块(子模块)
    ```
    若在父模块中定义了一个本地依赖`<systemPath>${project.build.directory}/lib</systemPath>`,该依赖在父模块中`${project.build.directory}`的路径会被解析成父模块的绝对路径,而在子模块中则会被处理成子模块的绝对路径,那么构建子模块时会提示找不到本地依赖
- 变量和pom
    - pom本身是独立的,maven本身就是对每个pom文件进行处理构建的过程
    - `${}`变量在不同的pom会表现出不同的值
- 正确处理本地依赖的方式
    - 使用私服是最好的
    - 将lib放置到需要改依赖的pom模块中,不要放到父级;使用`dependency: copy-dependencies`能完成打包
    
## 常见插件
### dependency插件
- `dependency:list`:打印项目所有的依赖
- `dependency:copy-dependencies -DoutputDirectory=xx`:拷贝依赖到指定路径
    - 打包本地依赖
        ```xml
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copyLocal</id>
                        <phase>compile</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}/${build.finalName}/WEB-INF/lib</outputDirectory>
                            <includeScope>system</includeScope>
                        </configuration>
                    </execution>
                </executions>
        ```