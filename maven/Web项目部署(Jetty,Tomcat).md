###Web项目部署到Servlet容器

####1.Summary

Web项目需要部署到Servlet容器中才可以工作。在maven项目中，我们可以将项目部署到jetty和tomcat这样的Servlet容器中。以下将记录详细配置。

####2.Jetty配置
#####2.1 运行：mvn jetty:run

    <plugins>
      <plugin>
        <groupId>org.mortbay.jetty</groupId>
        <artifactId>jetty-maven-plugin</artifactId>
        <configuration>

         <!-- 在很短指定的时间内扫描web应用检查是否有改变，如果发觉有任何改
              变则自动热部署。默认为0，表示禁用热部署检查。
              任何一个大于0的数字都将表示启用 -->
          <scanIntervalSeconds>0</scanIntervalSeconds>
          <war>${basedir}/target/${project.build.finalName}.war</war>
          <connectors>
            <!-- 更改jetty的端口，在pom中的配置中通过指定新的connector来实现，
                 例如下述 -->
            <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">
            <port>9090</port>
           </connector>
        </connectors>
        </configuration>
      </plugin>


#####2.2 更改jetty端口
可以使用Java属性：mvn -Djetty.port=9999 jetty:run

#####参考链接：
[jetty插件配置指南](http://www.blogjava.net/Jdonee/archive/2008/12/11/245650.html)


####3.Tomcat配置

Tomcat6,7配置稍有些差别，下面将分别进行说明

#####3.1 Tomcat6配置


######1. 添加Tomcat用户
添加一个用户，该用户的角色为：manager-gui，manager-script，manager-jmx和   manager-status，admin-gui，admin-scritp；修改${TOMCAT_HOME}/conf/tomcat-users.xml文件:

    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <role rolename="admin-gui"/>
    <role rolename="admin-scritp"/>
  
    <user username="tomcat6" password="123456" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>

######2. 修改项目的pom文件

    <build>
     <plugins>
      <plugin>
       <groupId>org.codehaus.mojo</groupId> 
       <artifactId>tomcat-maven-plugin</artifactId> 
       <version>1.1</version> 
       <configuration>
        <url>http://localhost:8080/manager/</url>
        <server>XINGJL</server>
        <username>tomcat6</username>  
        <password>123456</password>  
      </configuration>
      </plugin>    
    </plugins>
    </build>

######3. 部署项目
使用如下命令运行：
<pre>
mvn package tomcat:deploy
mvn package tomcat:redeploy
</pre>

#####3.2 Tomcat7配置

######1. 添加Tomcat用户

添加一个用户，该用户的角色为：manager-gui，manager-script，manager-jmx和   manager-status；修改${TOMCAT_HOME}/conf/tomcat-users.xml文件:

	<role rolename="manager-gui"/>
	<role rolename="manager-script"/>
	<role rolename="manager-jmx"/>
	<role rolename="manager-status"/>
  
	<user username="tomcat7" password="123456" roles="manager-gui,manager-status,manager-script,manager-jmx"/>
   
######2. 修改项目的pom文件


    <build>
     <plugins>
      <plugin>
       <groupId>org.apache.tomcat.maven</groupId>
       <artifactId>tomcat7-maven-plugin</artifactId>
       <version>2.1</version>
       <configuration>
           <uriEncoding>UTF-8</uriEncoding>
           <url>http://localhost:8080/manager/text</url>
           <username>tomcat7</username>  
           <password>123456</password>
       </configuration>
      </plugin> 
    </plugins>
    </build>

######3. 部署项目
使用如下命令运行：
<pre>
mvn package tomcat7:deploy
mvn package tomcat7:redeploy
</pre>


以上。