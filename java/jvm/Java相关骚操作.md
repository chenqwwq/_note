# Java 相关的非常规操作



## Java 动态编译

Java 的动态编译就是在运行时使用相关编译工具，动态的加载并编译相关源代码的技术。

相关类如下：

1. JavaCompiler（Java 编译的总入口，通过以下方式就可以获取到对应对象，JDK11之后具体的实现类变为了 JavacTool
2. JavaFileManager