---
title: solr入门
date: 2018-03-26 15:09:16
updated: 2018-03-26 15:09:16
categories: java
tags: [java, solr, 搜索]
comments: false
description: 你对本页的描述
photos:
  - http://wx3.sinaimg.cn/mw690/e2d4a00bgy1fuudol49nij21kw11zkcu.jpg
---

## solr原理

solr是一个独立的 **企业级搜索应用服务器**，它对外提供类似于 **web-service** 的api接口。用户可以通过http请求，向搜索引擎服务器提交一定格式的xml文件，生成索引。也可以通过http get操作提出查询的请求，得到xml/json格式的返回结果.

与elasticSearch比较:

|ElasticSearch|solr|
|:---:|:---:|
|IO非阻塞|IO阻塞|
|实时搜索(搜索的同时更新索引)较快|离线搜索(搜索的同时不更新索引)较快|
|社区相较不活跃|社区活跃|
|自身支持分布式搭建|搭建分布式需配合zookeeper|


Lucene是一个高效的，基于Java的 **全文检索库**。

所以在了解Lucene之前要费一番工夫了解一下全文检索。

那么什么叫做全文检索呢？这要从我们生活中的数据说起。

我们生活中的数据总体分为两种：结构化数据和非结构化数据。

**结构化数据** ：指具有固定格式或有限长度的数据，如数据库，元数据等。

**非结构化数据** ：指不定长或无固定格式的数据，如邮件，word文档等。

当然有的地方还会提到第三种，**半结构化数据**，如XML，HTML等，当根据需要可按结构化数据来处理，也可抽取出纯文本按非结构化数据来处理。

非结构化数据又一种叫法叫 **全文数据**。

按照数据的分类，搜索也分为两种：

对结构化数据的搜索 ：如对数据库的搜索，用SQL语句。再如对元数据的搜索，如利用windows搜索对文件名，类型，修改时间进行搜索等。

对非结构化数据的搜索 ：如利用windows的搜索也可以搜索文件内容，Linux下的grep命令，再如用Google和百度可以搜索大量内容数据。

对非结构化数据也即对全文数据的搜索主要有两种方法：

一种是 **顺序扫描法(Serial Scanning)** ：所谓顺序扫描，比如要找内容包含某一个字符串的文件，就是一个文档一个文档的看，对于每一个文档，从头看到尾，如果此文档包含此字符串，则此文档为我们要找的文件，接着看下一个文件，直到扫描完所有的文件。如利用windows的搜索也可以搜索文件内容，只是相当的慢。如果你有一个80G硬盘，如果想在上面找到一个内容包含某字符串的文件，不花他几个小时，怕是做不到。Linux下的grep命令也是这一种方式。大家可能觉得这种方法比较原始，但对于小数据量的文件，这种方法还是最直接，最方便的。但是对于大量的文件，这种方法就很慢了。

有人可能会说，对非结构化数据顺序扫描很慢，对结构化数据的搜索却相对较快（由于结构化数据有一定的结构可以采取一定的搜索算法加快速度），**那么把我们的非结构化数据想办法弄得有一定结构不就行了吗？**

这种想法很天然，却构成了全文检索的基本思路，也即将非结构化数据中的一部分信息提取出来，重新组织，使其变得有一定结构，然后对此有一定结构的数据进行搜索，从而达到搜索相对较快的目的。

这部分从 **非结构化数据中提取出的然后重新组织的信息**，我们称之 **索引**。

这种说法比较抽象，举几个例子就很容易明白，比如字典，字典的拼音表和部首检字表就相当于字典的索引，对每一个字的解释是非结构化的，如果字典没有音节表和部首检字表，在茫茫辞海中找一个字只能顺序扫描。然而字的某些信息可以提取出来进行结构化处理，比如读音，就比较结构化，分声母和韵母，分别只有几种可以一一列举，于是将读音拿出来按一定的顺序排列，每一项读音都指向此字的详细解释的页数。我们搜索时按结构化的拼音搜到读音，然后按其指向的页数，便可找到我们的非结构化数据——也即对字的解释。

 这种先建立索引，在对索引进行搜索的过程就叫做全文检索。

全文检索大体分为2个过程，索引创建和搜索索引

 1. 索引创建：将现实世界中的所有结构化和非结构化数据提取信息，创建索引的过程

 2. 搜索索引:就是得到用户查询的请求，搜索创建的索引，然后返回结果的过程

--------------

 于是全文检索就存在3个重要的问题:

 1. 索引里面究竟存了什么东西?

 2. 如何创建索引？

 3. 如何对索引进行搜索?
-------------

下面我们顺序对每个问题进行研究。
 

### 索引里面究竟存些什么

索引里面究竟需要存些什么呢？首先我们来看为什么顺序扫描的速度慢：其实是由于我们想要搜索的信息和非结构化数据中所存储的信息不一致造成的。

非结构化数据中所存储的信息是每个文件包含哪些字符串，也即已知文件，欲求字符串，也即是从文件到字符串的映射。而我们想搜索的信息是哪些文件包含此字符串，也即已知字符串，欲求文件，也即从字符串到文件的映射。两者恰恰相反。

于是如果索引总能够保存从字符串到文件的映射，则会大大提高搜索速度。

由于从字符串到文件的映射是文件到字符串映射的反向过程，于是保存这种信息的索引称为 **反向索引**。

反向索引的所保存的信息一般如下：

假设我的文档集合里面有100篇文档，为了方便表示，我们为文档编号从1到100，得到下面的结构

![](pic/solr1.jpg)


左边保存的是一系列字符串，称为词典。

每个字符串都指向包含此字符串的文档(Document)链表，此文档链表称为倒排表(Posting List)。

有了索引，便使保存的信息和要搜索的信息一致，可以大大加快搜索的速度。

比如说，我们要寻找既包含字符串“lucene”又包含字符串“solr”的文档，我们只需要以下几步：

1. 取出包含字符串“lucene”的文档链表。

2. 取出包含字符串“solr”的文档链表。

3. 通过合并链表，找出既包含“lucene”又包含“solr”的文件。

![](pic/solr2.jpg)


看到这个地方，有人可能会说，全文检索的确加快了搜索的速度，但是多了索引的过程，两者加起来不一定比顺序扫描快多少。的确，加上索引的过程，全文检索不一定比顺序扫描快，尤其是在数据量小的时候更是如此。而对一个很大量的数据创建索引也是一个很慢的过程。

然而两者还是有区别的，顺序扫描是每次都要扫描，而创建索引的过程仅仅需要一次，以后便是一劳永逸的了，每次搜索，创建索引的过程不必经过，仅仅搜索创建好的索引就可以了。

这也是全文搜索相对于顺序扫描的优势之一：一次索引，多次使用。

----

### 如何创建索引

全文检索的索引创建过程一般有以下几步：

**第一步**：一些要索引的原文档(Document)。

为了方便说明索引创建过程，这里特意用两个文件为例：

文件一：Students should be allowed to go out with their friends, but not allowed to drink beer.

文件二：My friend Jerry went to school to see his students but found them drunk which is not allowed.

 
**第二步**：将原文档传给分词组件(Tokenizer)。

分词组件(Tokenizer)会做以下几件事情(此过程称为Tokenize)：

1. 将文档分成一个一个单独的单词。

2. 去除标点符号。

3. 去除停词(Stop word)。

所谓停词(Stop word)就是一种语言中最普通的一些单词，由于没有特别的意义，因而大多数情况下不能成为搜索的关键词，因而创建索引时，这种词会被去掉而减少索引的大小。

英语中挺词(Stop word)如：“the”,“a”，“this”等。

对于每一种语言的分词组件(Tokenizer)，都有一个停词(stop word)集合。

经过分词(Tokenizer)后得到的结果称为词元(Token)。

在我们的例子中，便得到以下词元(Token)：

“Students”，“allowed”，“go”，“their”，“friends”，“allowed”，“drink”，“beer”，“My”，“friend”，“Jerry”，“went”，“school”，“see”，“his”，“students”，“found”，“them”，“drunk”，“allowed”。

**第三步**：将得到的词元(Token)传给语言处理组件(Linguistic Processor)。

语言处理组件(linguistic processor)主要是对得到的词元(Token)做一些同语言相关的处理。

对于英语，语言处理组件(Linguistic Processor)一般做以下几点：

1. 变为小写(Lowercase)。

2. 将单词缩减为词根形式，如“cars”到“car”等。这种操作称为：stemming。

3. 将单词转变为词根形式，如“drove”到“drive”等。这种操作称为：lemmatization

**第四步**：将得到的词(Term)传给索引组件(Indexer)。

1. 索引过程：

    a) 有一系列被索引文件

    b) 被索引文件经过语法分析和语言处理形成一系列词(Term)。

    c) 经过索引创建形成词典和反向索引表。

    d) 通过索引存储将索引写入硬盘。

2. 搜索过程：

    a) 用户输入查询语句。

    b) 对查询语句经过语法分析和语言分析得到一系列词(Term)。

    c) 通过语法分析得到一个查询树。

    d) 通过索引存储将索引读入到内存。

    e) 利用查询树搜索索引，从而得到每个词(Term)的文档链表，对文档链表进行交，差，并得到结果文档。

    f) 将搜索到的结果文档对查询的相关性进行排序。

    g) 返回查询结果给用户。

----

## solr使用入门

### 安装

1. 安装 Tomcat，解压缩即可。

2. 解压 solr。

3. 把 solr 下的 dist 目录 solr-4.10.3.war 部署到 Tomcat\webapps 下(去掉版本号)。

4. 启动 Tomcat 解压缩 war 包

5. 把solr下example/lib/ext 目录下的所有的 jar 包，添加到 solr 的工程中(\WEB-INF\lib目录下)。

6. 创建一个 solrhome 。solr 下的/example/solr 目录就是一个 solrhome。复制此目录
到 D 盘改名为 solrhome

7. 关联 solr 及 solrhome。需要修改 solr 工程的 web.xml 文件。

```xml
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>d:\solrhome</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry> 
```

8. 启动 Tomcat http://IP:8080/solr/

### IK分词器配置

1. 把 IKAnalyzer2012FF_u1.jar 添加到 solr 工程的 lib 目录下

2. 创建 WEB-INF/classes 文件夹 把扩展词典、停用词词典、配置文件放到 solr 工程
的 WEB-INF/classes 目录下。

3. 修改 Solrhome 的 schema.xml 文件，配置一个 FieldType，使用 IKAnalyzer

```xml
<fieldType name="text_ik" class="solr.TextField">
    <analyzer class="org.wltea.analyzer.lucene.IKAnalyzer"/>
</fieldType>
```

### 配置域

域相当于数据库的表字段，用户存放数据，因此用户根据业务需要去定义相关的 Field
（域），一般来说，每一种对应着一种数据，用户对同一种数据进行相同的操作。

域的常用属性：

-  name：指定域的名称
-  type：指定域的类型
-  indexed：是否索引
-  stored：是否存储
-  required：是否必须
-  multiValued：是否多值


1. 域
修改 solrhome 的 schema.xml 文件 设置业务系统 Field
```xml
<field name="item_goodsid" type="long" indexed="true" stored="true"/>
<field name="item_title" type="text_ik" indexed="true" stored="true"/>
<field name="item_price" type="double" indexed="true" stored="true"/>
<field name="item_image" type="string" indexed="false" stored="true" />
<field name="item_category" type="string" indexed="true" stored="true" />
<field name="item_seller" type="text_ik" indexed="true" stored="true" />
<field name="item_brand" type="string" indexed="true" stored="true" />
```

2. 复制域
复制域的作用在于将某一个 Field 中的数据复制到另一个域中
```xml
<field  name="item_keywords"  type="text_ik"  indexed="true"  stored="false"multiValued="true"/>
<copyField source="item_title" dest="item_keywords"/>
<copyField source="item_category" dest="item_keywords"/>
<copyField source="item_seller" dest="item_keywords"/>
<copyField source="item_brand" dest="item_keywords"/>
```

3. 动态域
当我们需要动态扩充字段时，我们需要使用动态域。对于某些特定字段的值是不确定的，
所以我们需要使用动态域来实现。需要实现的效果如下：
```xml
<dynamicField name="item_spec_*" type="string" indexed="true" stored="true"/>
```

### spring data solr

maven依赖:

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-solr</artifactId>
    <version>1.5.5.RELEASE</version>
</dependency>
```

基本配置(spring配置文件):
```xml
<!-- solr 服务器地址 -->
<solr:solr-server id="solrServer" url="http://127.0.0.1:8080/solr" />
<!-- solr 模板，使用 solr 模板可对索引库进行 CRUD 的操作 -->
<bean id="solrTemplate"class="org.springframework.data.solr.core.SolrTemplate">
    <constructor-arg ref="solrServer" />
</bean>
```

在实体类字段上添加注解@Field("索引库中字段名称"),如果没有指定字段名称,则跟实体类字段名一致.

最后调用solrTemplate进行增删改查操作.

## solr使用过程中的问题及解决方案



## solr部署(集群)