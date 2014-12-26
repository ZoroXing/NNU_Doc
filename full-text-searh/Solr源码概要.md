### Solr源码概要
----

####1.  总体结构

Solr程序是作为WebApp部署到像 Tomcat这样的容器中的。每一个Solr WebApp对应着一个Solr.home；该目录的结构如下：
<pre>
├─bin
├─collection1           <--- 一个core，其中collection1为core的名称。
│  ├─conf               <--- 存放该core配置文件的目录，最重要的是solrconfig和schema文件       
│  │  ├─solrconfig.xml  <--- 对该core的整体配置
│  │  ├─...
│  │  └─schema.xml      <--- 定义索引结构，filed和filedtype
│  ├─data               <--- 存放索引的目录  
│  └─core.properties    <--- 标志该目录为core的文件，其中name属性定义了core的名称。
│      
├─zoo.cfg
└─solr.xml              <--- 在非集群模式下，忽略该配置；里面定义了集群模式下的一些高级选项。 
</pre>

solr.home的配置有四种方式：

1. 在web.xml文件中：

    <env-entry>
     <env-entry-name>solr/home</env-entry-name>
     <env-entry-value>D:\solr\solr-home(solr.home的路径)</env-entry-value>
     <env-entry-type>java.lang.String</env-entry-type>
    </env-entry>

2. 通过Tomcat的JNDI方式（**JNDI: via java:comp/env/solr/home**）:在tomcat安装路径下：conf/Catalina/localhost目录下新建任意一个xml文件，例:solr.xml


    <Context docBase="webapp的部署路径(the_path_to solr.war)" debug="0" crossContext="true" >
       <Environment name="solr/home" type="java.lang.String" value="C:/example2/solr（the_path_to_solr_home）" override="true" />
    </Context>


3. tomcat启动JAVA_OPTS参数中：在tomcat的根目录下，找到bin/catalina.bat 在JAVA_OPTS选项中添加，如windows下,可在最前面加入一行:
<pre>
set JAVA_OPTS -Dsolr.solr.home=C:/example2/solr
</pre>

4. 使用程序当前目录

源码参见**org.apache.solr.core.SolrResourceLoader#locateSolrHome**
       

####2. 对Solr的请求

用户对Solr应用程序的请求都会被一个名为：SolrRequestFilter的拦截器拦截，定义如下：

    <filter>
      <filter-name>SolrRequestFilter</filter-name>
      <filter-class>org.apache.solr.servlet.SolrDispatchFilter</filter-class>
    </filter>
     
    <filter-mapping>
      <filter-name>SolrRequestFilter</filter-name>
      <url-pattern>/*</url-pattern>
    </filter-mapping>


**Solr请求：**

1. /admin/cores 

	该请求由CoreAdminHandler处理，获取当前所有core的信息 
2. /admin/collections

	该请求由CollectionsHandler处理，仅在Cloud模式下使用，获取当前集群信息
3. /admin/info

	该请求由InfoHandler处理
4. /[core_name]/schema
    
	- 如果省略<core_name>，则会使用默认**collection1**作为core 名称
	- 获取core的schema配置信息
	
5. //[core_name]/[requestHandle_name]

	- 如果省略<core_name>，则会使用默认**collection1**作为core 名称
    - 更handler名称交由该core下的requestHandler处理，如：
    	- select
    	- dataimport
    	- ......


以上。

