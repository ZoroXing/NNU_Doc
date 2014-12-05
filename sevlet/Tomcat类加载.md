##Tomcat类加载机制
<pre>
                                                                                          作成者：XINGJL
                                                                                          作成日：2014/12/04
</pre>
### 1. 概述

Tomcat内置了一系列的类加载器，保证运行在容器中的不同Web应用程序使用不同的类路径和资源仓库。在Java环境中，类加载机制采用双亲委派机制；但是，在Tomcat中WebappClassLoader加载器并未采用这种机制（在后面将详细讨论）。
随着Tomcat版本的不断变化，Tomcat5，Tomcat6和Tomcat7,8在类加载的过程中存在一些不同之处，在后面章节将逐一讨论。

### 2. Tomcat中的类加载器

Tomcat中自定义的类加载器有4种，分别是Common，Catalina，Shared
和WebApp，具体定义如下：

名称    |  实现类  |  父加载器  |  ClassPath
--------| -------- | ---------  | ---------
Common | org.apache.catalina.loader.StandardClassLoader | system | /common/*
Catalina| 同上 | common | /server/*
Shared  | 同上 | common | /shared/*
WebApp  | org.apache.catalina.loader.WebappClassLoader | Shared | /WebApp/WEB-INF/*

详细的关系如下图（pic2-1）：<br><br>
![Classloader parent-child relationships 5](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/tomcat/clsloader_5.5.png)

### 3. 类加载器初始化过程
Tomcat启动后首先启动初始化：Common,Catalina和Shared类加载器。具体调用如下：<br>
org.apache.catalina.startup.Bootstrap#main<br>
↓<br>
org.apache.catalina.startup.Bootstrap#init<br>
↓<br>
org.apache.catalina.startup.Bootstrap#initClassLoaders<br>
```
private void initClassLoaders() {
        try {
            // Common类加载器
            commonLoader = createClassLoader("common", null);
            if( commonLoader == null ) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader=this.getClass().getClassLoader();
            }
            // Catalina类加载器
            catalinaLoader = createClassLoader("server", commonLoader);
            // Shared类加载器
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
    }
```
org.apache.catalina.startup.Bootstrap#createClassLoader代码如下：
```
private ClassLoader createClassLoader(String name, ClassLoader parent)
        throws Exception {
        // 根据${Tomcat_home}\conf\catalina.properties获取属性值
        String value = CatalinaProperties.getProperty(name + ".loader");
        if ((value == null) || (value.equals("")))
            return parent;
-中略-
```
### 4. Tomcat各个版本类加载器区别
#### Tomcat5.5
  Tomcat5的中默认catalina.properties属性文件如下：
```
common.loader=${catalina.home}/common/classes,${catalina.home}/common/i18n/*.jar,${catalina.home}/common/endorsed/*.jar,${catalina.home}/common/lib/*.jar
server.loader=${catalina.home}/server/classes,${catalina.home}/server/lib/*.jar
shared.loader=${catalina.base}/shared/classes,${catalina.base}/shared/lib/*.jar
```
  因此，Tomcat5 会分别创建：Common,Catalina和Shared类加载器，相应关系如（pic2-1）
![Classloader parent-child relationships 5](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/tomcat/clsloader_5.5.png)
#### Tomcat6.x or later
默认catalina.properties属性文件如下：
```
common.loader=${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar
server.loader=
shared.loader=
```
因此，他们创建的Common,Catalina和Shared类加载器为同一个实例，即Common，相应的关系图如下(pic4-1)<br><br>
![Classloader parent-child relationships 6 or later](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/tomcat/clsloader_6_later.png)

### 5. Tomcat的类加载机制
Tomcat 的类加载器实现类有两种：StandardClassLoader和WebappClassLoader两种。其中StandardClassLoader未重载loadClass方法，因此采用的仍然是双亲委派原则；然而WebappClassLoader重载了loadClass方法，使用了另外一种类加载策略。然而，各个版本又有稍许差别。

#### Tomcat6
##### 当delegate="false"时，默认值
1. 检查缓存中是否存在要加载的类
2. 若缓存中没有，则首先使用**系统类加载器**加载，防止Web应用程序中的类覆盖J2SE的类
3. 从当前仓库中载入相关类
4. 使用父类载入器。若父类载入器为null，使用**系统的类加载器**进行加载
5. 若仍未找到需要的类，则抛出ClasNotFoundException异常。<br>
类加载顺序:<br>

-  Bootstrap classes of your JVM
-  System class loader classes (described above)
-  /WEB-INF/classes of your web application
-  /WEB-INF/lib/*.jar of your web application
-  $CATALINA_HOME/common/classes
-  $CATALINA_HOME/common/endorsed/*.jar
-  $CATALINA_HOME/common/i18n/*.jar
-  $CATALINA_HOME/common/lib/*.jar
-  $CATALINA_BASE/shared/classes
-  $CATALINA_BASE/shared/lib/*.jar

##### 当delegate="true"时
1. 检查缓存中是否存在要加载的类
2. 若缓存中没有，则首先使用**系统类加载器**加载，防止Web应用程序中的类覆盖J2SE的类
3. 使用父类载入器。若父类载入器为null，使用**系统的类加载器**进行加载
4. 从当前仓库中载入相关类
5. 若仍未找到需要的类，则抛出ClasNotFoundException异常。<br>
类加载顺序:<br>

-  Bootstrap classes of your JVM
-  System class loader classes (described above)
-  $CATALINA_HOME/common/classes
-  $CATALINA_HOME/common/endorsed/*.jar
-  $CATALINA_HOME/common/i18n/*.jar
-  $CATALINA_HOME/common/lib/*.jar
-  $CATALINA_BASE/shared/classes
-  $CATALINA_BASE/shared/lib/*.jar
-  /WEB-INF/classes of your web application
-  /WEB-INF/lib/*.jar of your web application

**备注**：默认情况下，Tomcat6 shared类加载器路径。

org.apache.catalina.loader.WebappClassLoader#loadClass<br>
```
public synchronized Class loadClass(String name, boolean resolve)
        throws ClassNotFoundException {

        if (log.isDebugEnabled())
            log.debug("loadClass(" + name + ", " + resolve + ")");
        Class clazz = null;

        // Log access to stopped classloader
        if (!started) {
            try {
                throw new IllegalStateException();
            } catch (IllegalStateException e) {
                log.info(sm.getString("webappClassLoader.stopped", name), e);
            }
        }

        // (0) Check our previously loaded local class cache
        clazz = findLoadedClass0(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }

        // (0.1) Check our previously loaded class cache
        clazz = findLoadedClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }

        // (0.2) Try loading the class with the system class loader, to prevent
        //       the webapp from overriding J2SE classes
        try {
            // 使用系统类加载器
            clazz = system.loadClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // (0.5) Permission to access this class when using a SecurityManager
        if (securityManager != null) {
            int i = name.lastIndexOf('.');
            if (i >= 0) {
                try {
                    securityManager.checkPackageAccess(name.substring(0,i));
                } catch (SecurityException se) {
                    String error = "Security Violation, attempt to use " +
                        "Restricted Class: " + name;
                    log.info(error, se);
                    throw new ClassNotFoundException(error, se);
                }
            }
        }

        boolean delegateLoad = delegate || filter(name);

        // (1) Delegate to our parent if requested
        if (delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader1 " + parent);
            ClassLoader loader = parent;
            if (loader == null)
                loader = system;
            try {
                clazz = loader.loadClass(name);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                ;
            }
        }

        // (2) Search local repositories
        if (log.isDebugEnabled())
            log.debug("  Searching local repositories");
        try {
            clazz = findClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from local repository");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            ;
        }

        // (3) Delegate to parent unconditionally
        if (!delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader at end: " + parent);
            ClassLoader loader = parent;
            if (loader == null)
                loader = system;
            try {
                clazz = loader.loadClass(name);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                ;
            }
        }

        throw new ClassNotFoundException(name);

    }
```

#### Tomcat7 or later

##### 当delegate="false"时，默认值
1. 检查缓存中是否存在要加载的类
2. 若缓存中没有，则首先使用**引导类加载器**加载，防止Web应用程序中的类覆盖J2SE的类
3. 从当前仓库中载入相关类
4. 使用父类载入器。若父类载入器为null，使用**引导类加载器**进行加载
5. 若仍未找到需要的类，则抛出ClasNotFoundException异常。<br>

类加载顺序:<br>
-  Bootstrap classes of your JVM
-  /WEB-INF/classes of your web application
-  /WEB-INF/lib/*.jar of your web application
-  System class loader classes (described above)
-  Common class loader classes (described above)

##### 当delegate="true"时
1. 检查缓存中是否存在要加载的类
2. 若缓存中没有，则首先使用**引导类加载器**加载，防止Web应用程序中的类覆盖J2SE的类
3. 使用父类载入器。若父类载入器为null，使用**引导类加载器**进行加载
4. 从当前仓库中载入相关类
5. 若仍未找到需要的类，则抛出ClasNotFoundException异常。<br>
类加载顺序:<br>

-  Bootstrap classes of your JVM
-  System class loader classes (described above)
-  Common class loader classes (described above)
-  /WEB-INF/classes of your web application
-  /WEB-INF/lib/*.jar of your web application

org.apache.catalina.loader.WebappClassLoader#loadClass<br>
```
    public synchronized Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {

        if (log.isDebugEnabled())
            log.debug("loadClass(" + name + ", " + resolve + ")");
        Class<?> clazz = null;

        // Log access to stopped classloader
        if (!started) {
            try {
                throw new IllegalStateException();
            } catch (IllegalStateException e) {
                log.info(sm.getString("webappClassLoader.stopped", name), e);
            }
        }

        // (0) Check our previously loaded local class cache
        clazz = findLoadedClass0(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }

        // (0.1) Check our previously loaded class cache
        clazz = findLoadedClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }

        // (0.2) Try loading the class with the system class loader, to prevent
        //       the webapp from overriding J2SE classes
        try {
            // j2seClassLoader为引导类加载器。
            clazz = j2seClassLoader.loadClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // (0.5) Permission to access this class when using a SecurityManager
        if (securityManager != null) {
            int i = name.lastIndexOf('.');
            if (i >= 0) {
                try {
                    securityManager.checkPackageAccess(name.substring(0,i));
                } catch (SecurityException se) {
                    String error = "Security Violation, attempt to use " +
                        "Restricted Class: " + name;
                    log.info(error, se);
                    throw new ClassNotFoundException(error, se);
                }
            }
        }

        boolean delegateLoad = delegate || filter(name);

        // (1) Delegate to our parent if requested
        if (delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader1 " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }

        // (2) Search local repositories
        if (log.isDebugEnabled())
            log.debug("  Searching local repositories");
        try {
            clazz = findClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from local repository");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // (3) Delegate to parent unconditionally
        if (!delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader at end: " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }

        throw new ClassNotFoundException(name);

    }
```
j2seClassLoader类的初始化<br>
```
public WebappClassLoader() {
-中略-
        ClassLoader j = String.class.getClassLoader();
        if (j == null) {
            j = getSystemClassLoader();
            while (j.getParent() != null) {
                j = j.getParent();
            }
        }
        this.j2seClassLoader = j;
-中略-
}
```

### 参考链接
[1].[Tomcat5.5 Class Loader HOW-TO](http://tomcat.apache.org/tomcat-5.5-doc/class-loader-howto.html)<br>
[2].[Tomcat6.0 Class Loader HOW-TO](http://tomcat.apache.org/tomcat-6.0-doc/class-loader-howto.html)<br>
[3].[Tomcat7.0 Class Loader HOW-TO](http://tomcat.apache.org/tomcat-7.0-doc/class-loader-howto.html)<br>
[4].[Tomcat8.0 Class Loader HOW-TO](http://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html)<br>
[5].[Tomcat download](http://archive.apache.org/dist/tomcat/)<br>

以上。

