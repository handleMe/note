# Maven 构建 Java 项目

Maven 使用原型 **archetype** 插件创建项目。要创建一个简单的 Java 应用，我们将使用 **maven-archetype-quickstart** 插件。

在下面的例子中，我们将在 C:\MVN 文件夹下创建一个基于 maven 的 java 应用项目。

命令格式如下：

```
mvn archetype:generate "-DgroupId=com.companyname.bank" "-DartifactId=consumerBanking" "-DarchetypeArtifactId=maven-archetype-quickstart" "-DinteractiveMode=false"
```

参数说明：

- **-DgroupId**: 组织名，公司网址的反写 + 项目名称
- **-DartifactId**: 项目名-模块名
- **-DarchetypeArtifactId**: 指定 ArchetypeId，maven-archetype-quickstart，创建一个简单的 Java 应用
- **-DinteractiveMode**: 是否使用交互模式

# Maven 引入外部依赖

pom.xml 的 dependencies 列表列出了我们的项目需要构建的所有外部依赖项。

```HTML
<dependencies>
    <!-- 在这里添加你的依赖 -->
    <dependency>
        <groupId>ldapjdk</groupId>  <!-- 库名称，也可以自定义 -->
        <artifactId>ldapjdk</artifactId>    <!--库名称，也可以自定义-->
        <version>1.0</version> <!--版本号-->
        <scope>system</scope> <!--作用域-->
        <systemPath>${basedir}\src\lib\ldapjdk.jar</systemPath> <!--项目根目录下的lib文件夹下-->
    </dependency> 
</dependencies>
```

# Maven 项目文档

本章节我们主要学习如何创建 Maven 项目文档。

比如我们在 C:/MVN 目录下，创建了 consumerBanking 项目，Maven 使用下面的命令来快速创建 java 项目：

```
mvn archetype:generate -DgroupId=com.companyname.bank -DartifactId=consumerBanking -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

修改 pom.xml，添加以下配置（如果没有的话）：

```HTML
<project>
  ...
<build>
<pluginManagement>
    <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-site-plugin</artifactId>
          <version>3.3</version>
        </plugin>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-project-info-reports-plugin</artifactId>
          <version>2.7</version>
        </plugin>
    </plugins>
    </pluginManagement>
</build>
 ...
</project>
```

*不然运行* **mvn site** *命令时出现* **java.lang.NoClassDefFoundError: org/apache/maven/doxia/siterenderer/DocumentContent** *的问题， 这是由于 maven-site-plugin 版本过低，升级到 3.3+ 即可。*

打开 consumerBanking 文件夹并执行以下 mvn 命令。

```
C:\MVN\consumerBanking> mvn site
```

Maven 开始生成文档：

```
[INFO] Scanning for projects...
[INFO] -------------------------------------------------------------------
[INFO] Building consumerBanking
[INFO]task-segment: [site]
[INFO] -------------------------------------------------------------------
[INFO] [site:site {execution: default-site}]
[INFO] artifact org.apache.maven.skins:maven-default-skin: 
checking for updates from central
[INFO] Generating "About" report.
[INFO] Generating "Issue Tracking" report.
[INFO] Generating "Project Team" report.
[INFO] Generating "Dependencies" report.
[INFO] Generating "Continuous Integration" report.
[INFO] Generating "Source Repository" report.
[INFO] Generating "Project License" report.
[INFO] Generating "Mailing Lists" report.
[INFO] Generating "Plugin Management" report.
[INFO] Generating "Project Summary" report.
[INFO] -------------------------------------------------------------------
[INFO] BUILD SUCCESSFUL
[INFO] -------------------------------------------------------------------
[INFO] Total time: 16 seconds
[INFO] Finished at: Wed Jul 11 18:11:18 IST 2012
[INFO] Final Memory: 23M/148M
[INFO] -------------------------------------------------------------------
```

打开 **C:\MVN\consumerBanking\target\site** 文件夹。点击 **index.html** 就可以看到文档了。

Maven 使用一个名为 [Doxia](http://maven.apache.org/doxia/index.html)的文档处理引擎来创建文档，它能将多种格式的源码读取成一种通用的文档模型。要为你的项目撰写文档，你可以将内容写成下面几种常用的，可被 Doxia 转化的格式。

| 格式名 | 描述                     | 参考                                                     |
| :----- | :----------------------- | :------------------------------------------------------- |
| Apt    | 纯文本文档格式           | http://maven.apache.org/doxia/references/apt-format.html |
| Xdoc   | Maven 1.x 的一种文档格式 | http://jakarta.apache.org/site/jakarta-site2.html        |
| FML    | FAQ 文档适用             | http://maven.apache.org/doxia/references/fml-format.html |
| XHTML  | 可扩展的 HTML 文档       | http://en.wikipedia.org/wiki/XHTML                       |