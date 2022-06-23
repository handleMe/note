## 环境配置

Maven基于java，需要JDK环境

### 系统要求

| 项目     | 要求                                                         |
| :------- | :----------------------------------------------------------- |
| JDK      | Maven 3.3 要求 JDK 1.7 或以上 Maven 3.2 要求 JDK 1.6 或以上 Maven 3.0/3.1 要求 JDK 1.5 或以上 |
| 内存     | 没有最低要求                                                 |
| 磁盘     | Maven 自身安装需要大约 10 MB 空间。除此之外，额外的磁盘空间将用于你的本地 Maven 仓库。你本地仓库的大小取决于使用情况，但预期至少 500 MB |
| 操作系统 | 没有最低要求                                                 |

### 环境变量

- windows：新建系统变量 MAVEN_HOME，变量值：E:\Maven\apache-maven-3.3.9  编辑系统变量 Path，添加变量值：;%MAVEN_HOME%\bin

- linux：
  编辑 /etc/profile 文件 sudo vim /etc/profile，在文件末尾添加如下代码：

```
export MAVEN_HOME=/usr/local/apache-maven-3.3.9
export PATH=${PATH}:${MAVEN_HOME}/bin
```

保存文件，并运行如下命令使环境变量生效：
```# source /etc/profile```

在控制台输入如下命令，如果能看到 Maven 相关版本信息，则说明 Maven 已经安装成功：
```# mvn -v```

- mac：

编辑 **/etc/profile** 文件 **sudo vim /etc/profile**，在文件末尾添加如下代码：

```
export MAVEN_HOME=/usr/local/apache-maven-3.3.9
export PATH=${PATH}:${MAVEN_HOME}/bin
```

保存文件，并运行如下命令使环境变量生效：

```
$ source /etc/profile
```

在控制台输入如下命令，如果能看到 Maven 相关版本信息，则说明 Maven 已经安装成功：

```
$ mvn -v
```
