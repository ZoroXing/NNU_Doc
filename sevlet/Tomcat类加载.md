##Tomcat类加载机制

### 1. 概述

Tomcat内置了一系列的类加载器，保证运行在容器中的不同Web应用程序使用不同的类路径和资源仓库。在Java环境中，类加载机制采用双亲委派机制；但是，在Tomcat中WebappClassLoader加载器并未采用这种机制（在后面将详细讨论）。
随着Tomcat版本的不断变化，Tomcat5，Tomcat6和Tomcat7,8在类加载的过程中存在一些不同之处，在后面章节将逐一讨论。

### 2. Tomcat中的类加载器

Tomcat中自定义的类加载器有4种，分别是Common，Catalina，Shared
和WebApp，具体定义如下：

名称   | 实现类 | 父加载器 | ClassPath
Common |org.apache.catalina.loader.StandardClassLoader|System | /common/*
Catalina|同上|common|/server/*
Shared|同上|common | 