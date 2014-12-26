#### Solr索引数据库配置

##### 1. 配置solrconfig.xml

修改core中的solrconfig.xml配置文件，添加requestHandler。例如：${Solr.home}/collection1/conf/solrconfig.xml

     <requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">  
        <lst name="defaults">  
            <str name="config">data-config.xml</str>  
        </lst>  
    </requestHandler>  

#####2. 使用RDBMS作为索引对象

RDBMS的相关信息配置在data-config.xml中，主要用途如下：

1.  How to fetch data (queries,url etc)
2.  What to read ( resultset columns, xml fields etc)
3.  How to process (modify/add/remove fields)

---------------------------------------------------------------------------------------------
    <dataConfig>
     <dataSource type="JdbcDataSource"
              driver="com.mysql.jdbc.Driver"
              url="jdbc:mysql://10.10.21.14:3306/chain_base"
              user="root"
              password="cluster@yonyou.com" />
      <document>
        <entity name="solr_test" transformer="DateFormatTransformer"
            query="SELECT id, consignee, address, mobile,date FROM base_address">
            <field column='date' dateTimeFormat='yyyy-MM-dd HH:mm:ss' />
        </entity>
    </document>
    </dataConfig>
---------------------------------------------------------------------------------------------

1.  document：一个文档也就是lucene的document，一个文档包含一个或多个根entity。

2.  entity： 一个entity代表关系数据库中的一个table/view,一个根entity可以包含多个子entity或多个field。
	1. entity默认属性
	
		- name：一个唯一的名称用于区分不同entity（随意）
		- processor：默认为SqlEntityProcessor，其他非RDBMS的需要明确指定；
		- transformer：
		- dataSource：仅当有多个dataSources时使用；
		- threads：运行该entity所需线程的数量；
		- pk：entity的主键，仅仅在delta-imports中使用，与schema.xml中的uniquekey无关；
		- rootEntity：boolean值，确定该entity是否为根entity；
		- onError:(abort|skip|continue)，当发生error时，如何处理；
		- preImportDeleteQuery：在full-import命令执行前，需要清除index。默认为“*：*”，这里可以自定义；
		- postImportDeleteQuery：
	2. SqlEntityProcessor特有属性：
		- query：查询数据库使用的sql语句；获取full-import需要的数据SQL.<br>**工作原理:**<br>执行本Entity的Query，获取所有数据；针对每个行数据Row，获取pk，组装子Entity的Query；执行子Entity的Query，获取子Entity的数据。
		- deltaImportQuery:获取delta-import需要的数据的SQL<br>**工作原理:**<br>查找子Entity，直到没有为止；执行Entity的deltaQuery，获取变化数据的pk；合并子Entity parentDeltaQuery得到的pk；针对每一个pk Row，组装父Entity的parentDeltaQuery；执行parentDeltaQuery，获取父Entity的pk；执行deltaImportQuery，获取自身的数据；如果没有deltaImportQuery，就组装Query<br>**限制:**
			- 子Entity的query必须引用父Entity的pk
			- 子Entity的parentDeltaQuery必须引用自己的pk
			- 子Entity的parentDeltaQuery必须返回父Entity的pk 
			- deltaImportQuery引用的必须是自己的pk
		- deltaQuery：获取当前entity中更新记录的主键的SQL
		- parentDeltaQuery：获取父Entity中更新记录的主键的SQL
		- deletePkQuery:
		

3.  field：属性column是数据库的字段，name是filed的名字，即schema中的field name。假如column name与field name 不同，则需要手动指定field name属性。 


#####3. 索引数据库

使用如下URL可以索引数据库

- full-import
	1. 该操作会还开起一个新线程，并且_status_ 属性变为_busy now_
	2. 该操作会花费一些时间，依赖于dataset的大小；
	3. 操作执行时，会储存操作开始的时间到文件_conf/dataimport.properties_,该时间戳用于delta-import操作中；
	4. 在full-import执行期，查询不会被阻塞；
	5. 其他的一些属性：
		- entity：选择索引的一个或多个entity的名字，默认索引doc下全部entity；
		- clean：（default为‘true’）全索引前是否清除已有索引；
		- commit:（default为'true'）操作后是否进行commit；
		- optimize：（default 'true' up to Solr 3.6, 'false' afterwards）是否进行优化，耗时操作；
		- debug：（default 'false'）是否运行在debug模式，该模式下不会立即commit，需要显示指定'commit=true'

  <pre>
  http://localhost:8080/Solr-4.7.0/dataimport?command=full-import
  http://localhost:8080/Solr-4.7.0/dataimport?command=full-import&clean=false
  </pre>  

- detla-import： 增量导入（增量索引）支持full-import_其他的一些属性_；
  <pre>
  http://localhost:8080/Solr-4.7.0/dataimport?command=delta-import
  注： 本例暂时未实现
  </pre>

- status：查看当前命令的状态，给出详细的统计：
	- 多少doc被创建
	- 多少doc被删除
	- 多少doc被查询
	- 多少doc rows fetched

  <pre>
  http://<host>:<port>/solr/dataimport
  </pre>

- reload-config ：假如_data-config.xml_配置文件改变需重新加载（无需重启Solr），可以运行如下命令：
  <pre>
  http://<host>:<port>/solr/dataimport?command=reload-config
  </pre>
- abort : 终止一个进行中的操作
  <pre>
  http://<host>:<port>/solr/dataimport?command=abort
  </pre>

#####4. 参考链接

1. [Wiki:DataImportHandler](http://wiki.apache.org/solr/DataImportHandler)
2. [利用SOLR搭建企业搜索平台](http://blog.csdn.net/duck_genuine/article/details/5459137)

以上。

   


