#### Solr整合中文分词mmseg4j


##### 1. Summary
本次使用的是Solr-4.7.0整合mmseg4j-1.9.1，下载mmseg4j-1.9.1.zip，把dist下面的jar文件拷贝到${solr-4.7.0}/WEB-INF/lib中，共三个jar文件

- mmseg4j-analysis-1.9.1.jar，
- mmseg4j-core-1.9.1.jar
- mmseg4j-solr-1.9.1.jar

##### 2. 修改core中配置文件schema.xml

在core的配置文件（schema.xml）中，添加使用中文分词器的fieldtype。例如：${Solr.home}/collection1/conf/schema.xml

    <!-- mmseg4j-->  
    <fieldType name="text_complex" class="solr.TextField" positionIncrementGap="100" >    
        <analyzer>    
            <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="complex" dicPath="dic"/>    
        </analyzer>    
    </fieldType>    
    <fieldType name="text_maxword" class="solr.TextField" positionIncrementGap="100" >    
        <analyzer>    
            <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="max-word" dicPath="dic"/>    
        </analyzer>    
    </fieldType>    
    <fieldType name="text_simple" class="solr.TextField" positionIncrementGap="100" >    
        <analyzer>    
          <!--  
            <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="simple" dicPath="n:/OpenSource/apache-solr-1.3.0/example/solr/my_dic"/>   
            -->  
            <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="simple" dicPath="dic"/>       
        </analyzer>    
    </fieldType>  
    <!-- mmseg4j--> 


mmseg4j使用的是MMSeg算法，算法有两种分词方法：Simple和Complex，都是基于正向最大匹配。Complex 加了四个规则过虑。官方说：词语的正确识别率达到了 98.41%。mmseg4j 已经实现了这两种分词算法。

上例中添加了三中fieldtype，他们主要区别是使用的模式不同：

-  text_simple   使用Simple分词方法
-  text_complex  Complex 加了四个规则过虑
-  text_maxword  默认。在complex基础上实现了最多分词(max-word)。“很好听” -> "很好|好听"; “中华人民共和国” -> "中华|华人|共和|国"; “中国人民银行” -> "中国|人民|银行"。 

##### 3. 源码修改

运行服务器，检验中文分词。当输入‘中华人民共和国’并单机“Analyse Values”按钮时，出现如下错误：

![](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/solr/mmseg4j-exception.jpg)

修改mmseg4j-analysis模块的源码文件：com.chenlb.mmseg4j.analysis.MMSegTokenizer增加如下一行代码：

<pre>
	public void reset() throws IOException {
		//lucene 4.0
		//org.apache.lucene.analysis.Tokenizer.setReader(Reader)
		//setReader 自动被调用, input 自动被设置。
+		super.reset();
		mmSeg.reset(input);
	}
</pre>

保存后，重新编译打包：

<pre>
mvn clean package -DskipTests
</pre>

成功运行后如下：

![](https://github.com/ZoroXing/NNU_Doc/blob/master/picture/solr/mmseg4j-normal.jpg)