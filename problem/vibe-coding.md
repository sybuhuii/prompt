你这次遇到的不是代码问题，而是构建环境没有暴露给 KSCC 的命令行环境。

实际遇到的是什么问题

你的环境是：

Windows
├── IDEA 中配置了 JDK
├── IDEA 自带 Maven 3.9.11
└── Windows 全局 PATH 中原本没有 java 和 mvn

IDEA 能正常编译，是因为 IDEA 自己知道：

JDK：
C:\Users\KC\.jdks\openjdk-26.0.1

Maven：
C:\Program Files\JetBrains\IntelliJ IDEA 2026.1.4
\plugins\maven\lib\maven3\bin\mvn.cmd

但 KSCC 启动的执行环境主要是 Bash，它只能通过系统环境变量和命令路径寻找：

java
mvn

当它执行：

java -version
mvn -version

时找不到命令，于是开始：

扫描磁盘目录；
查询 Windows 注册表；
查找 .jdks；
尝试 cmd.exe；
尝试 Bash 路径；
失败后自动重试 10 次。

所以看起来像 KSCC 一直“卡住”，其实它是在错误的执行环境里反复寻找构建工具。

长期最好的解决办法
方案一：给项目添加 Maven Wrapper

以后项目根目录存在：

mvnw
mvnw.cmd
.mvn/

任何工具都可以直接执行：

.\mvnw.cmd -DskipTests compile

不需要猜 Maven 安装位置。这是最适合多人开发、CI 和 AI 编程工具的方案。

不过仍然需要系统能找到 Java，或者明确设置 JAVA_HOME。

方案二：永久配置 Windows 环境变量

配置：

JAVA_HOME=C:\Users\KC\.jdks\openjdk-26.0.1

并把下面路径加入 Path：

%JAVA_HOME%\bin

Maven则安装一个独立版本，再配置：

MAVEN_HOME=C:\apache-maven-3.9.11

把下面路径加入 Path：

%MAVEN_HOME%\bin

之后新终端中应该能直接运行：

java -version
mvn -version

这种方式对所有命令行 AI 工具最省事。

方案三：在 AGENTS.md 中固定记录构建命令

例如：

## Windows 构建环境

- 项目根目录：`C:\Users\KC\Desktop\agent-platform`
- JAVA_HOME：`C:\Users\KC\.jdks\openjdk-26.0.1`
- Maven：`C:\Program Files\JetBrains\IntelliJ IDEA 2026.1.4\plugins\maven\lib\maven3\bin\mvn.cmd`
- 不要扫描注册表或磁盘寻找 Java/Maven。
- 完成全部修改后只执行一次编译。

这样以后 KSCC、Claude Code 等工具读取 AGENTS.md 后，就不会从头猜测。

总结

这次完整的问题链路是：

IDEA 内部有 Java 和 Maven
        ↓
Windows 全局命令行看不到 Maven
        ↓
KSCC 在 Bash 中执行 java/mvn 失败
        ↓
KSCC 不断扫描路径和自动重试
        ↓
执行速度变慢并浪费上下文
        ↓
找到真实 JDK、Maven路径并明确告诉它
        ↓
可以直接编译，不再探测

所以它会影响那些需要调用 Java/Maven、但读取不到环境配置的 AI 工具；把环境变量配置好，或者引入 Maven Wrapper 后，这类问题基本就不会反复出现。


