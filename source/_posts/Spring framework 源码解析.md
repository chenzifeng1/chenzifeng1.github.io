# Spring framework

## 编译源码

spring源码git仓库地址: [https://github.com/spring-projects/spring-framework/](https://github.com/spring-projects/spring-framework/)

之后可以根据自己需求下载不同版本分支的代码

![Spring%20framework%2080df3c81d678438c85eb0f8059dafbf0/Untitled.png](Spring%20framework%2080df3c81d678438c85eb0f8059dafbf0/Untitled.png)

我们也可以去把整个项目clone之后，去checkout到对应的tag版本

`git clone [https://github.com/spring-projects/spring-framework.git](https://github.com/spring-projects/spring-framework.git)`

clone之后我们可以根据tag的不同版本号进行chackout，在本次源码学习中，我选择的 `v5.2.3.RELEASE` 的tag，chackout命令如下

`git checkout v5.2.3.RELEASE`

编译下载好的代码：

- 先编译oxm子项目，在文件中打开cmd 命令： `gradlew :spring-oxm:compileTestJava`
- 之后编译其他的项目，命令：`gradlew build`

编译好之后，打开项目，会发现每个项目底下的文件结构如下

![Spring%20framework%2080df3c81d678438c85eb0f8059dafbf0/Untitled%201.png](Spring%20framework%2080df3c81d678438c85eb0f8059dafbf0/Untitled%201.png)

**build:** build目录下包括编译之后的class文件，依赖libs，日志logs，资源文件resources，临时文件tmp等。

**src:**下面就是我们的源码了