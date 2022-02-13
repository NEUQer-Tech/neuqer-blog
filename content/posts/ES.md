---
title: "ES"
date: 2022-02-13T18:54:43+08:00
draft: false
authorLink: https://github.com/Henrik-Yao
tags: ["技术分享","ES"]
categories: ["ES"]
draft: false
---

@[TOC]
## 一.elasticsearch简介

> **Elasticsearch**是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于**RESTful web**接口。Elasticsearch是用Java语言开发的，并作为Apache许可条款下的开放源码发布，是一种流行的企业级搜索引擎。Elasticsearch用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。
> **Lucene**是Apache的开源搜索引擎类库，提供了搜索引擎的核心API
> 

> mysql采用正向索引（B树，B+树） 
>
> elasticsearch采用倒排索引

![请添加图片描述](https://img-blog.csdnimg.cn/4c55ecd9759d4090ab500ae961d2a34f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

> Mysql：擅长事务类型操作，可以确保数据的安全和一致性 
>
> Elasticsearch：擅长海量数据的搜索、分析、计算

**概念对比**

![在这里插入图片描述](https://img-blog.csdnimg.cn/8e9018fc46f54dc4b38384c48568ba2f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 二.docker部署es和kibana

> kibana可以给我们提供一个elasticsearch的可视化界面，便于学习

**1.创建互联网联，让es和kibana容器互联**

```sh
docker network create es-net
```
**2.拉取镜像**

```powershell
docker pull elasticsearch:7.12.1
docker pull kibana:7.12.1
```
**3.部署单点es**

```powershell
docker run -d \
	--name es \
    -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "discovery.type=single-node" \
    -v es-data:/usr/share/elasticsearch/data \
    -v es-plugins:/usr/share/elasticsearch/plugins \
    --privileged \
    --network es-net \
    -p 9200:9200 \
    -p 9300:9300 \
elasticsearch:7.12.1
```
命令解释：

> - `-e "cluster.name=es-docker-cluster"`：设置集群名称
> - `-e "http.host=0.0.0.0"`：监听的地址，可以外网访问
> - `-e "ES_JAVA_OPTS=-Xms512m -Xmx512m"`：内存大小
> - `-e "discovery.type=single-node"`：非集群模式
> - `-v es-data:/usr/share/elasticsearch/data`：挂载逻辑卷，绑定es的数据目录
> - `-v es-logs:/usr/share/elasticsearch/logs`：挂载逻辑卷，绑定es的日志目录
> - `-v es-plugins:/usr/share/elasticsearch/plugins`：挂载逻辑卷，绑定es的插件目录
> - `--privileged`：授予逻辑卷访问权
> - `--network es-net` ：加入一个名为es-net的网络中
> - `-p 9200:9200`：端口映射配置

访问9200端口即可看到elasticsearch的响应结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/a43050df039d40f79e772714e46e2c42.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


**4.部署kibana**
```powershell
docker run -d \
--name kibana \
-e ELASTICSEARCH_HOSTS=http://es:9200 \
--network=es-net \
-p 5601:5601  \
kibana:7.12.1
```

> - `--network es-net` ：加入一个名为es-net的网络中，与elasticsearch在同一个网络中
> - `-e ELASTICSEARCH_HOSTS=http://es:9200"`：设置elasticsearch的地址，因为kibana已经与elasticsearch在一个网络，因此可以用容器名直接访问elasticsearch
> - `-p 5601:5601`：端口映射配置


访问5601端口即可看到kibana的响应结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/a23ae8e215704bdda933e1a7cb0745f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 三.IK分词器
es在创建倒排索引时需要对文档分词；在搜索时，需要对用户输入内容分词。但默认的分词规则对中文处理并不友好
处理中文分词，一般会使用IK分词器。https://github.com/medcl/elasticsearch-analysis-ik

**1.安装IK分词器**
```powershell
# 进入容器内部
docker exec -it elasticsearch /bin/bash

# 在线下载并安装
./bin/elasticsearch-plugin  install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.12.1/elasticsearch-analysis-ik-7.12.1.zip

#退出
exit
#重启容器
docker restart elasticsearch
```

**2.IK分词器包含两种模式**：

* `ik_smart`：智能切分 最少切分 粗粒度 分出的词较少

* `ik_max_word`：最细切分 细粒度 分出的词较多 内存消耗高

**3.拓展词库**
要拓展ik分词器的词库，只需要修改一个ik分词器目录中的config目录中的IkAnalyzer.cfg.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 *** 添加扩展词典-->
        <entry key="ext_dict">ext.dic</entry>
</properties>
```
然后在名为ext.dic的文件中，添加想要拓展的词语即可

**4.停用词库**
要禁用某些敏感词条，只需要修改一个ik分词器目录中的config目录中的IkAnalyzer.cfg.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典-->
        <entry key="ext_dict">ext.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典  *** 添加停用词词典-->
        <entry key="ext_stopwords">stopword.dic</entry>
</properties>
```
然后在名为stopword.dic的文件中，添加想要拓展的词语即可

## 四.DSL及Dev Tools

> 官网学习地址：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

DSL是elasticsearch提供的JSON风格的请求语句，用来操作elasticsearch，实现CRUD

```json
GET /  相当于直接访问9200端口
```


Dev Tools是kibana提供的一种可视化工具

![在这里插入图片描述](https://img-blog.csdnimg.cn/58e0828f72d1472580e33de68989ebd6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 五.索引库操作
**1.mapping属性**

映射是定义文档及其包含的字段如何存储和索引的过程。  
每个文档都是字段的集合，每个字段都有自己的数据类型。 在映射数据时，创建一个映射定义，该定义包含与文档相关的字段列表。 映射定义还包括元数据字段，比如_source字段，它自定义如何处理文档的相关元数据。  
mapping是对索引库中文档的约束，常见的mapping属性包括：

 - type：字段数据类型，常见的简单类型有：
    - 字符串：text（可分词的文本）、keyword（精确值，例如：品牌、国家、ip地址）
   - 数值：long、integer、short、byte、double、float、
   - 布尔：boolean
   - 日期：date
   - 对象：object
 - index：是否创建索引，默认为true
 - analyzer：使用哪种分词器
 - properties：该字段的子字段

**2.创建索引库**

ES中通过Restful请求操作索引库、文档。请求内容用DSL语句来表示。创建索引库和mapping的DSL语法如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/c494a175488545a5bfa42812e9d34e4a.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/5180e2d60568446298ebf9593325965c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**3.查询索引库**

```xml
GET /索引库名
```


![在这里插入图片描述](https://img-blog.csdnimg.cn/0fad02dcd681487dbad399384873a81c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


**4.删除索引库**
```xml
DELETE /索引库名
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/9182362b0494445dbf44ab2c05ffbf07.png)


**5.修改索引库**

索引库和mapping一旦创建无法修改，但是可以添加新的字段，语法如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/9ac68904d44e4fa98e63a762469563ef.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/07ae5e269abc42f3b7c07e624a86ca68.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 五.文档操作
**1.添加文档**

![在这里插入图片描述](https://img-blog.csdnimg.cn/3d66c920ccde42969c6ec62a0b366b45.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/82916308d5d64c0cb133a0df4b5b78d3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)**2.查询文档**

```xml
GET /索引库名/_doc/文档id
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/fd049d4a0f50428883f7f55d438f8f3a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**3.删除文档**

```xml
DELETE /索引库名/_doc/文档id
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/4d8c28f16f87464fbdc86b8f0d2ba7ec.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**4.修改文档**

 - 方式一：全量修改，会删除旧文档，添加新文档

![在这里插入图片描述](https://img-blog.csdnimg.cn/d99666d552434dfdafa87b401149bc2e.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/1fe5e1a6668948d8907c9a9f1299d3e2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

 - 方式二：增量修改，修改指定字段值

![在这里插入图片描述](https://img-blog.csdnimg.cn/a49cf74c9c6a4206ae78e8cac1f7bf3b.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/26da7bbd83d7449e801a15e694129520.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 六.RestClient操作索引库
ES官方提供了各种不同语言的客户端，用来操作ES。这些客户端的本质就是组装DSL语句，通过http请求发送给ES

官方文档地址：https://www.elastic.co/guide/en/elasticsearch/client/index.html

**1.初始化RestClient**
指定版本，需要与es版本一致

```xml
<properties>
     <java.version>1.8</java.version>
     <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```
导入包

```xml
<dependency>
     <groupId>org.elasticsearch.client</groupId>
     <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```
初始化RestClient

```java
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(
	HttpHost.create("http://101.43.16.42:9200")
));;
```
**2.创建索引库**

```java
@Test
void createHotelIndex() throws IOException {
    // 1.创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("hotel");
    // 2.准备请求参数；DSL语句
    //MAPPING_TEMPLATE是静态常量字符串，内容是创建索引库的DSL语句
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // 3.发送请求
    client.indices().create(request, RequestOptions.DEFAULT);
}
```
**3.删除索引库**

```java
@Test
void testDeleteHotelIndex() throws IOException {
    // 1.创建Request对象
    DeleteIndexRequest request = new DeleteIndexRequest("hotel");
    // 2.发送请求
    client.indices().delete(request, RequestOptions.DEFAULT);
}
```
**4.判断索引库是否存在**

```java
@Test
void testExitHotelIndex() throws IOException {
    // 1.创建Request对象
    GetIndexRequest request = new GetIndexRequest("hotel");
    // 2.发送请求
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
    //3.输出结果
    System.out.println(exists);
}
```
## 七.RestClient操作文档
**1.新增文档**

```java
 @Test
void testAddDocument() throws IOException {
    //根据id查询酒店数据
    Hotel hotel = hotelService.getById(61083L);
    // 1.创建Request对象
    IndexRequest request = new IndexRequest("hotel").id(hotel.getId().toString());
    // 2.准备请求参数；DSL语句
    request.source(JSON.toJSONString(hotel),XContentType.JSON);
    // 3.发送请求
    client.index(request, RequestOptions.DEFAULT);
}
```
**2.查询文档**

```java
@Test
void testGetDocumentById() throws IOException {
    // 1.创建Request对象
    GetRequest request = new GetRequest("hotel", "61083");
    // 2.发送请求
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 3.解析响应结果
    String json = response.getSourceAsString();
    //反序列化
    HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
    System.out.println(hotelDoc);
}
```
**3.更新文档**

```java
@Test
void testUpdateDocument() throws IOException {
    // 1.准备request
    UpdateRequest request = new UpdateRequest("hotel", "61083");
    // 2.准备请求参数
    request.doc(
            "price","952",
            "starName","四钻"
    );
    // 3.发送请求
    client.update(request, RequestOptions.DEFAULT);
}
```
**4.删除文档**

```java
@Test
void testDeleteDocument() throws IOException {
    // 1.准备request
    DeleteRequest request = new DeleteRequest("hotel", "61083");
    // 2.发送请求
    client.delete(request, RequestOptions.DEFAULT);
}
```
**5.批量新增文档**

```java
@Test
void testBulkDocument() throws IOException {
    //批量查询酒店数据
    List<Hotel> hotels = hotelService.list();
    // 1.创建Request对象
    BulkRequest request = new BulkRequest();
    // 2.准备请求参数，添加多个新增的Request
    for(Hotel hotel:hotels){
        // 转换为HotelDoc
        HotelDoc hotelDoc = new HotelDoc(hotel);
        request.add(new IndexRequest("hotel")
                .id(hotel.getId().toString())
                .source(JSON.toJSONString(hotelDoc),XContentType.JSON));
    }
    // 3.发送请求
    client.bulk(request, RequestOptions.DEFAULT);
}
```
## 八.DSL查询语法

> Elasticsearch提供了基于JSON的DSL（Domain Specific  Language）来定义查询。
> 官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html

常见的查询类型包括：

 - 查询所有：查询出所有数据，一般测试用。例如：
   - match_all
 - 全文检索（full text）查询：利用分词器对用户输入内容分词，然后去倒排索引库中匹配。例如：
   - match_query
   - multi_match_query
 - 精确查询：根据精确词条值查找数据，一般是查找keyword、数值、日期、boolean等类型字段。例如：
   - ids
   - range
   - term
 - 地理（geo）查询：根据经纬度查询。例如：
   - geo_distance
   - geo_bounding_box
 - 复合（compound）查询：复合查询可以将上述各种查询条件组合起来，合并查询条件。例如：
   - bool
   - function_score

查询的语法基本一致： 

![在这里插入图片描述](https://img-blog.csdnimg.cn/62c010a66de94f628d48f19e1d80b662.png)




- 查询类型为match_all
- 没有查询条件
![在这里插入图片描述](https://img-blog.csdnimg.cn/b0355bab574f43299a4bf7305dd7bea8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**1.全文检索**

全文检索查询，会对输入框输入内容分词，常用于搜索框搜索

①match查询：单字段查询
![在这里插入图片描述](https://img-blog.csdnimg.cn/86e40aea563245aea6633bfcac8f2753.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

②multi_match查询：多字段查询，任意一个字段符合条件就算符合查询条件

![在这里插入图片描述](https://img-blog.csdnimg.cn/2f7aad5da08b4635b188578222ce23b1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

> ps：multi_match根据多个字段查询，参与查询字段越多，查询性能越差，使用copy_to将多字段拷贝到一个字段中可以提升性能

**2.精确查询**

精确查询一般是查找keyword、数值、日期、boolean等类型字段。所以**不会**对搜索条件分词

①term：根据词条精确值查询

> 因为精确查询的字段搜是不分词的字段，因此查询的条件也必须是不分词的词条。查询时，用户输入的内容跟自动值完全匹配时才认为符合条件。如果用户输入的内容过多，反而搜索不到数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/385611676ddd45cf9244295556cdd477.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

②range：根据值的范围查询

> gte代表大于等于，gt则代表大于
>  lte代表小于等于，lt则代表小于

![在这里插入图片描述](https://img-blog.csdnimg.cn/bc393174346c416992c688e79decd9b1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**3.地理查询**

①geo_distance
附近查询，也叫做距离查询（geo_distance）：查询到指定中心点小于某个距离值的所有文档
![在这里插入图片描述](https://img-blog.csdnimg.cn/0901f881e3564ed5b2e0faad6d510dde.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

②geo_bounding_box
矩形范围查询，也就是geo_bounding_box查询，查询坐标落在某个矩形范围的所有文档
![在这里插入图片描述](https://img-blog.csdnimg.cn/f2e78c1b45ce47c787dc0f5b8773ce06.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_11,color_FFFFFF,t_70,g_se,x_16)

**4.复合查询**

复合（compound）查询：复合查询可以将其它简单查询组合起来，实现更复杂的搜索逻辑。常见的有两种：

- fuction score：算分函数查询，可以控制文档相关性算分，控制文档排名
- bool query：布尔查询，利用逻辑关系组合多个其它的查询，实现复杂搜索

**相关性算法**

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd1141bfc98f4dd69725589b4da972a0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**TF对比BM25**

![在这里插入图片描述](https://img-blog.csdnimg.cn/00ab812f05824fa4b3555fd5051ee38e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)①fuction score

function score query定义的三要素

 - 过滤条件：哪些文档要加分
 - 算分函数：如何计算function score
 - 加权方式：function score 与 query score如何运算![请添加图片描述](https://img-blog.csdnimg.cn/7341f4e2d44a40c7b8a517b03766270d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)![在这里插入图片描述](https://img-blog.csdnimg.cn/b0d2c42d0e624b43a10e6bc132cb2bd8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
 
②bool query

布尔查询是一个或多个查询子句的组合，每一个子句就是一个子查询。子查询的组合方式有：
 - must：必须匹配每个子查询，类似“与”
 - should：选择性匹配子查询，类似“或”
 - must_not：必须不匹配，不参与算分，类似“非”
 - filter：必须匹配，不参与算分


![在这里插入图片描述](https://img-blog.csdnimg.cn/0e5eb2ba81b144e6b1723c2e45446ae0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

> 需要注意的是，搜索时，参与打分的字段越多，查询的性能也越差。因此这种多条件查询时，一遍这样做：
> 
> - 搜索框的关键字搜索，是全文检索查询，使用must查询，参与算分 
> - 其它过滤条件，采用filter查询。不参与算分

## 九.搜索结果处理

elasticsearch默认是根据相关度算分（_score）来排序，但是也支持自定义方式对搜索结果排序。可以排序字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等。

官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html

**1.排序**


①常规字段排序

![在这里插入图片描述](https://img-blog.csdnimg.cn/69deca557c964f9c852339aa53311fed.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_16,color_FFFFFF,t_70,g_se,x_16)![在这里插入图片描述](https://img-blog.csdnimg.cn/3aa5768b81844066ab9f36b69a286491.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
②地理位置排序

![在这里插入图片描述](https://img-blog.csdnimg.cn/d43ca7f601bd48d5a2bf5d43b15e9cc8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_9,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/943bca2ef37a45c181d3a3acf198abe5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**2.分页**

![在这里插入图片描述](https://img-blog.csdnimg.cn/d14b7d74e6bd48ceaea6493f270dce46.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**深度分页问题**

![在这里插入图片描述](https://img-blog.csdnimg.cn/2d704c2cbca545ccb8c177a60dda34f2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/0e1ee62578284cb9ac5e43288fed83c8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**解决深度分页问题**

官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html

- search after：分页时需要排序，原理是从上一次的排序值开始，查询下一页数据。官方推荐使用的方式。
- scroll：原理将排序后的文档id形成快照，保存在内存。官方已经不推荐使用。

**分页查询的常见实现方案以及优缺点**

- `from + size`：
  - 优点：支持随机翻页
  - 缺点：深度分页问题，默认查询上限（from + size）是10000
  - 场景：百度、京东、谷歌、淘宝这样的随机翻页搜索
- `after search`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：只能向后逐页查询，不支持随机翻页
  - 场景：没有随机翻页需求的搜索，例如手机向下滚动翻页

- `scroll`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：会有额外内存消耗，并且搜索结果是非实时的
  - 场景：海量数据的获取和迁移。从ES7.1开始不推荐，建议用 after search方案。

**3.高亮**

高亮显示的实现分为两步：
给文档中的所有关键字都添加一个标签，例如<em>标签
页面给<em>标签编写CSS样式

![在这里插入图片描述](https://img-blog.csdnimg.cn/3801c8a9b4954d11839a294de520a30a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_13,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/779db707f92345cf92133601274383aa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 十.RestClient查询文档

> 查询的基本步骤是：
> 
> 1. 创建SearchRequest对象
> 
> 2. 准备Request.source()，也就是DSL。
> 
>    ① QueryBuilders来构建查询条件
> 
>    ② 传入Request.source() 的 query() 方法
> 
> 3. 发送请求，得到结果
> 
> 4. 解析结果（参考JSON结果，从外到内，逐层解析）

![在这里插入图片描述](https://img-blog.csdnimg.cn/29c4ca567ac8476f91af9ebeb09b37a7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/b1f6bbf46565416c888a47a6de558d16.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
elasticsearch返回的结果是一个JSON字符串，结构包含：

- `hits`：命中的结果
  - `total`：总条数，其中的value是具体的总条数值
  - `max_score`：所有结果中得分最高的文档的相关性算分
  - `hits`：搜索结果的文档数组，其中的每个文档都是一个json对象
    - `_source`：文档中的原始数据，也是json对象

因此，解析响应结果，就是逐层解析JSON字符串，流程如下：

- `SearchHits`：通过response.getHits()获取，就是JSON中的最外层的hits，代表命中的结果
  - `SearchHits#getTotalHits().value`：获取总条数信息
  - `SearchHits#getHits()`：获取SearchHit数组，也就是文档数组
    - `SearchHit#getSourceAsString()`：获取文档结果中的_source，也就是原始的json文档数据


完整代码

```java
@Test
void testMatchAll() throws IOException {
    // 1.创建Request对象
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备请求参数；DSL语句
    request.source().query(QueryBuilders.matchAllQuery());
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);

    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获得总条数
    long total = searchHits.getTotalHits().value;
    System.out.println(total);
    // 4.1.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for(SearchHit hit : hits){
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        System.out.println(hotelDoc);
    }
}
```

**1.全文检索查询**

![在这里插入图片描述](https://img-blog.csdnimg.cn/82373657445444f99b2b7efbe6efbeed.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)



```java
@Test
void testMatch() throws IOException {
    // 1.创建Request对象
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备请求参数；DSL语句
    request.source()
            .query(QueryBuilders.matchQuery("all", "如家"));
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);

    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获得总条数
    long total = searchHits.getTotalHits().value;
    System.out.println(total);
    // 4.1.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for(SearchHit hit : hits){
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        System.out.println(hotelDoc);
    }
}
```

Ctrl+Alt+M可以抽取重复代码

```java
private void handleResponse(SearchResponse response) {
    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获得总条数
    long total = searchHits.getTotalHits().value;
    System.out.println(total);
    // 4.1.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for (SearchHit hit : hits) {
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        System.out.println(hotelDoc);
    }
}
```

**2.精确查询**

![在这里插入图片描述](https://img-blog.csdnimg.cn/9385f6a9e8ea46569680deafc8f62ad7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**3.复合查询**

![请添加图片描述](https://img-blog.csdnimg.cn/07993a34fb144eb68d6cfcdcd633a1fa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/804b48c9658b4604aa5e7239c43cb2d7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
@Test
void testBool() throws IOException {
    // 1.创建Request对象
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备请求参数；DSL语句
    // 2.1.准备BooleanQuery
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 2.2.添加term
    boolQuery.must(QueryBuilders.termQuery("city","上海"));
    // 2.3.添加range
    boolQuery.filter(QueryBuilders.rangeQuery("price").gte(100));
    request.source().query(boolQuery);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    handleResponse(response);
}
```
**4.排序和分页**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20a589166cfd407b8f8bb755b61b3202.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
@Test
void testPageAndSort() throws IOException {
    // 页码，每页大小
    int page = 2, size = 5;
    // 1.创建Request对象
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备请求参数；DSL语句
    // 2.1.query
    request.source().query(QueryBuilders.matchAllQuery());
    // 2.2.sort
    request.source().sort("price", SortOrder.ASC);
    // 2.3.分页
    request.source().from((page-1)*size).size(size);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    handleResponse(response);
}
```

距离排序

![请添加图片描述](https://img-blog.csdnimg.cn/6dceffc9ba094f5aa03f6bd248029f16.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


**5.高亮**

![请添加图片描述](https://img-blog.csdnimg.cn/62ae6484db674e83870a47e7f3cd615a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
@Test
void testHighlight() throws IOException {
    // 1.创建Request对象
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备请求参数；DSL语句
    // 2.1.query
    request.source().query(QueryBuilders.matchQuery("all", "如家"));
    // 2.2.高亮
    request.source().highlighter(new HighlightBuilder().field("name").requireFieldMatch(false));
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    handleResponse(response);
}
```

高亮结果解析
![请添加图片描述](https://img-blog.csdnimg.cn/69169aa9fadc4aaababda543c303b8e0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
重写解析方法

```java
private void handleResponse(SearchResponse response) {
    // 4.解析响应
    SearchHits searchHits = response.getHits();
    // 4.1.获得总条数
    long total = searchHits.getTotalHits().value;
    System.out.println(total);
    // 4.1.文档数组
    SearchHit[] hits = searchHits.getHits();
    // 4.3.遍历
    for (SearchHit hit : hits) {
        // 获取文档source
        String json = hit.getSourceAsString();
        // 反序列化
        HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
        // 获取高亮结果
        Map<String, HighlightField> highlightFields = hit.getHighlightFields();
        if(!CollectionUtils.isEmpty(highlightFields)){
            // 根据字段名获取高亮结果
            HighlightField highlightField = highlightFields.get("name");
            // 获取高亮值
            String name = highlightField.getFragments()[0].string();
            // 覆盖非高亮结果
            hotelDoc.setName(name);
        }
        // 覆盖非高亮值
        System.out.println(hotelDoc);
    }
}
```
## 十一.数据聚合
聚合是对文档数据的统计、分析、计算

参与聚合的字段类型必须是：keyword，数值，日期，布尔

聚合常见的有三类：

- 桶（Bucket）聚合：用来对文档做分组
  - TermAggregation：按照文档字段值分组，例如按照品牌值分组、按照国家分组
  - Date Histogram：按照日期阶梯分组，例如一周为一组，或者一月为一组

- 度量（Metric）聚合：用以计算一些值，比如：最大值、最小值、平均值等
  - Avg：求平均值
  - Max：求最大值
  - Min：求最小值
  - Stats：同时求max、min、avg、sum等
- 管道（pipeline）聚合：其它聚合的结果为基础做聚合

**1.桶（Bucket）聚合**

![在这里插入图片描述](https://img-blog.csdnimg.cn/656d869010134790be40f62750a861ef.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_18,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/5c7e174389f9443e93e2111ada84eb99.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
默认情况下，Bucket聚合会统计Bucket内的文档数量，记为_count，并且按照_count降序排序

可以指定order属性，自定义聚合的排序方式

![在这里插入图片描述](https://img-blog.csdnimg.cn/a2091c4b81234932b9c09626c2a0f282.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_12,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/6949ea80d9b442dc94e72c81e6ed738d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
限定聚合范围

![在这里插入图片描述](https://img-blog.csdnimg.cn/ec3a01e5896e4d6eacf357c4451f74b4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_11,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2ac556c373d9465590d074daf644e069.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)



**2. 度量（Metric）聚合**

![在这里插入图片描述](https://img-blog.csdnimg.cn/49d0192d966b4c4bb8d5219e2978b472.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_14,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/26b2927cb3dc4809bef3bf65e39733ed.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 十二.RestClient数据聚合

![请添加图片描述](https://img-blog.csdnimg.cn/81e24f4bdad541f0be8ca76f50d025bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

![请添加图片描述](https://img-blog.csdnimg.cn/09bf3e511c404a08abe11dcd38eaf01b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
@Test
void testAggregation() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    request.source().size(0);
    request.source().aggregation(AggregationBuilders
            .terms("brandAgg")
            .field("brand")
            .size(10)
    );
    // 3.发出请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析结果
    Aggregations aggregations = response.getAggregations();
    // 4.1.根据聚合名称获取聚合结果
    Terms brandTerms = aggregations.get("brandAgg");
    // 4.2.获取buckets
    List<? extends Terms.Bucket> buckets = brandTerms.getBuckets();
    // 4.3.遍历
    for (Terms.Bucket bucket : buckets){
        // 4.4.获取key
        String key = bucket.getKeyAsString();
        System.out.println(key);
    }
}
```

## 十三.自动补全
**1.拼音分词器**

要实现根据字母做补全，就必须对文档按照拼音分词。
在GitHub上有elasticsearch的拼音分词插件。
地址：https://github.com/medcl/elasticsearch-analysis-pinyin

**2.自定义分词器**

elasticsearch中分词器（analyzer）的组成包含三部分：

character filters：在tokenizer之前对文本进行处理。例如删除字符、替换字符
tokenizer：将文本按照一定的规则切割成词条（term）。例如keyword，就是不分词；还有ik_smart
tokenizer filter：将tokenizer输出的词条做进一步处理。例如大小写转换、同义词处理、拼音处理等

![请添加图片描述](https://img-blog.csdnimg.cn/f0db1f22b74a4080b4ada12de9c9b508.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)参考官网配置

![在这里插入图片描述](https://img-blog.csdnimg.cn/447b0a3c66d84566b8af1211829f90e8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


自定义分词器语法如下

```json
PUT /test
{
  "settings": {
    "analysis": {
      "analyzer": { // 自定义分词器
        "my_analyzer": {  // 分词器名称
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": { // 自定义tokenizer filter
        "py": { // 过滤器名称
          "type": "pinyin", // 过滤器类型，这里是pinyin
		  "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

> 拼音分词器注意事项：
> 
> 为了避免搜索到同音字，搜索时不要使用拼音分词器

**3.自动补全**

elasticsearch提供了Completion Suggester查询来实现自动补全功能。这个查询会匹配以用户输入内容开头的词条并返回

官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-suggesters.html

为了提高补全查询的效率，对于文档中字段的类型有一些约束：

- 参与补全查询的字段必须是completion类型。

- 字段的内容一般是用来补全的多个词条形成的数组。

![在这里插入图片描述](https://img-blog.csdnimg.cn/bc3e628db00c4b94927c00198026a463.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/a7ad6301d1094c328d989ad17adc1fd2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_11,color_FFFFFF,t_70,g_se,x_16)
![请添加图片描述](https://img-blog.csdnimg.cn/1b81e618acf6431fb7956b7f2a25a03f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


![请添加图片描述](https://img-blog.csdnimg.cn/b9177a16918a4eb7b67276c292acc5b3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

```java
@Test
void testSuggestion() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    request.source().suggest(new SuggestBuilder().addSuggestion(
            "suggestion",
            SuggestBuilders.completionSuggestion("suggestion")
            .prefix("h")
            .skipDuplicates(true)
            .size(10)
    ));
    // 3.发起请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析结果
    Suggest suggest = response.getSuggest();
    // 4.1.根据补全查询名称，获取补全结果
    CompletionSuggestion suggestion = suggest.getSuggestion("suggestion");
    // 4.2.获取options
    List<CompletionSuggestion.Entry.Option> options = suggestion.getOptions();
    for(CompletionSuggestion.Entry.Option option : options){
        String text = option.getText().toString();
        System.out.println(text);
    }
}
```

## 十四.数据同步
elasticsearch中的酒店数据来自于mysql数据库，因此mysql数据发生改变时，elasticsearch也必须跟着改变，这个就是elasticsearch与mysql之间的数据同步

常见的数据同步方案有三种：
- 同步调用
- 异步通知
- 监听binlog

**1.方式一：同步调用**

- 优点：实现简单，粗暴
- 缺点：业务耦合度高

![请添加图片描述](https://img-blog.csdnimg.cn/f0d561df243141729fdcb79d880e3109.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)



**2.方式二：异步通知**

- 优点：低耦合，实现难度一般
- 缺点：依赖mq的可靠性

![请添加图片描述](https://img-blog.csdnimg.cn/6cb226ab0d904f8d8b110fde3dbf5134.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


**3.方式三：监听binlog**

- 优点：完全解除服务间耦合
- 缺点：开启binlog增加数据库负担、实现复杂度高

![请添加图片描述](https://img-blog.csdnimg.cn/7f38a1639f1943beb2f3cf33c3400175.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)


## 十五.ES集群
单机的elasticsearch做数据存储，必然面临两个问题：海量数据存储问题、单点故障问题。

- 海量数据存储问题：将索引库从逻辑上拆分为N个分片（shard），存储到多个节点
- 单点故障问题：将分片数据在不同节点备份（replica ）

数据备份可以保证高可用，但是每个分片备份一份，所需要的节点数量就会翻一倍，成本实在是太高了

为了在高可用和成本间寻求平衡，我们可以这样做：

- 首先对数据分片，存储到不同节点
- 然后对每个分片进行备份，放到对方节点，完成互相备份

这样可以大大减少所需要的服务节点数量

![请添加图片描述](https://img-blog.csdnimg.cn/8f6995ee34ca451c90567f5ec43eed9f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**1.部署es集群**

使用docker-compose

```docker
version: '2.2'
services:
  es01:
    image: elasticsearch:7.12.1
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: elasticsearch:7.12.1
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - data02:/usr/share/elasticsearch/data
    ports:
      - 9201:9200
    networks:
      - elastic
  es03:
    image: elasticsearch:7.12.1
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic
    ports:
      - 9202:9200
volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```

es运行需要修改一些linux系统权限，修改`/etc/sysctl.conf`文件

```sh
vi /etc/sysctl.conf
```

添加下面的内容：

```sh
vm.max_map_count=262144
```

然后执行命令，让配置生效：

```sh
sysctl -p
```



通过docker-compose启动集群：

```sh
docker-compose up -d
```

**2.集群状态监控**

kibana可以监控es集群，不过新版本需要依赖es的x-pack 功能，配置比较复杂。

这里使用cerebro来监控es集群状态

官方网址：https://github.com/lmenezes/cerebro

![在这里插入图片描述](https://img-blog.csdnimg.cn/18e32cec8bc34fb9885be7612349f4e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
启动后输入es地址即可监控

**3.创建索引库**

```json
PUT /itcast
{
  "settings": {
    "number_of_shards": 3, // 分片数量
    "number_of_replicas": 1 // 副本数量
  },
  "mappings": {
    "properties": {
      // mapping映射定义 ...
    }
  }
}
```
![请添加图片描述](https://img-blog.csdnimg.cn/ffee562441204861ace314bc64ca8cf6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**4.es集群节点角色**

![请添加图片描述](https://img-blog.csdnimg.cn/09d1eb282b17433e9464d0dc870b2e19.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
- master节点：对CPU要求高，但是内存要求低
- data节点：对CPU和内存要求都高
- coordinating节点：对网络带宽、CPU要求高


职责分离可以让我们根据不同节点的需求分配不同的硬件去部署。而且避免业务之间的互相干扰。

> master eligible节点
> 
> - 参与集群选主
> - 主节点可以管理集群状态、管理分片信息、处理创建和删除索引库的请求
> 
> data节点
> 
> - 数据的CRUD
> 
> coordinator节点
> 
> - 路由请求到其它节点
> 
> - 合并查询到的结果，返回给用户

一个典型的es集群职责划分如图：

![请添加图片描述](https://img-blog.csdnimg.cn/acfaedc5622a4f48b1e9bd565e343991.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**5.脑裂**

脑裂是因为集群中的节点失联导致的。

例如一个集群中，主节点与其它节点失联：

此时，node2和node3认为node1宕机，就会重新选主：

当node3当选后，集群继续对外提供服务，node2和node3自成集群，node1自成集群，两个集群数据不同步，出现数据差异。

当网络恢复后，因为集群中有两个master节点，集群状态的不一致，出现脑裂的情况：

解决脑裂的方案是，要求选票超过 ( eligible节点数量 + 1 ）/ 2 才能当选为主，因此eligible节点数量最好是奇数。对应配置项是discovery.zen.minimum_master_nodes，在es7.0以后，已经成为默认配置，因此一般不会发生脑裂问题

例如：3个节点形成的集群，选票必须超过 （3 + 1） / 2 ，也就是2票。node3得到node2和node3的选票，当选为主。node1只有自己1票，没有当选。集群中依然只有1个主节点，没有出现脑裂。

![请添加图片描述](https://img-blog.csdnimg.cn/8d6e18610f2547d9be0dc870be7154b4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
**6.分片存储原理**
elasticsearch会通过hash算法来计算文档应该存储到哪个分片：

![请添加图片描述](https://img-blog.csdnimg.cn/b24c1a70883d4112bde9241afa636f47.png)




说明：

- _routing默认是文档的id
- 算法与分片数量有关，因此索引库一旦创建，分片数量不能修改！





新增文档的流程如下：

![请添加图片描述](https://img-blog.csdnimg.cn/cfc54d36a66744c38316e4df20e8c234.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)




解读：

- 1）新增一个id=1的文档
- 2）对id做hash运算，假如得到的是2，则应该存储到shard-2
- 3）shard-2的主分片在node3节点，将数据路由到node3
- 4）保存文档
- 5）同步给shard-2的副本replica-2，在node2节点
- 6）返回结果给coordinating-node节点


**7.集群分布式查询**
elasticsearch的查询分成两个阶段：

- scatter phase：分散阶段，coordinating node会把请求分发到每一个分片

- gather phase：聚集阶段，coordinating node汇总data node的搜索结果，并处理为最终结果集返回给用户

![请添加图片描述](https://img-blog.csdnimg.cn/ffe0248d1c644783aad007a2520130a9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)

**8.集群故障转移**

集群的master节点会监控集群中的节点状态，如果发现有节点宕机，会立即将宕机节点的分片数据迁移到其它节点，确保数据安全，这个叫做故障转移

![请添加图片描述](https://img-blog.csdnimg.cn/5546094b04e74a93b79dc197c3d77217.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASGVucmlrLVlhbw==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 十六.附录
#### 1.相关代码
github仓库：https://github.com/Henrik-Yao/Hotel-ES

#### 2.相关DSL

```json
GET /

GET _search
{
  "query": {
    "match_all": {}
  }
}


POST /_analyze
{
 "text": "宁可卷死自己，不让其他人休息",
 "analyzer": "ik_smart"
}


POST /_analyze
{
 "text": "宁可卷死自己，不让其他人休息",
 "analyzer": "ik_max_word"
}

POST /_analyze
{
 "text": "不洗碗工作室宁可卷死自己，不让其他人休息",
 "analyzer": "ik_smart"
}


PUT /test
{
  "mappings": {
    "properties": {
      "info":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email":{
        "type": "keyword",
        "index": false
      },
      "name":{
        "type": "object",
        "properties": {
          "firstName":{
            "type": "keyword",
            "index": false
          },
          "lastName":{
            "type": "keyword",
            "index": false
          }
        }
      }
    }
  }
}

GET /test


DELETE /test


PUT /test/_mapping
{
  "properties":{
    "age":{
      "type":"integer"
    }
  }
}


POST /test/_doc/1
{
  "info":"不洗碗工作室",
  "email":"henrik@qq.com",
  "name": {
    "firstName":"云",
    "lastName":"赵"
  }
}


GET /test/_doc/1



DELETE /test/_doc/1


PUT /test/_doc/1
{
  "info":"不洗碗工作室",
  "email":"henrik@qq.com",
  "name": {
    "firstName":"云",
    "lastName":"赵"
  }
}



POST /test/_update/1
{
  "doc":{
    "email":"henrik-yao@qq.com"
  }
}


GET /hotel


DELETE /hotel



GET /hotel/_doc/61083



GET /hotel/_search


GET /hotel/_search
{
  "query": {
    "match_all": {}
  }
}





GET /indexName/_search
{
  "query": {
    "查询类型": {
      "查询条件": "条件值"
    }
  }
}





GET /hotel/_search
{
  "query": {
    "match": {
      "all": "外滩如家"
    }
  }
}


DELETE /hotel



PUT /hotel
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "copy_to": "all"
      },
      "address": {
        "type": "keyword",
        "index": false
      },
      "price": {
        "type": "integer"
      },
      "score": {
        "type": "integer"
      },
      "brand": {
        "type": "keyword",
        "copy_to": "all"
      },
      "city": {
        "type": "keyword"
      },
      "starName": {
        "type": "keyword"
      },
      "business": {
        "type": "keyword",
        "copy_to": "all"
      },
      "pic": {
        "type": "keyword",
        "index": false
      },
      "location": {
        "type": "geo_point"
      },
      "all": {
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}



GET /hotel/_search
{
  "query": {
    "match": {
      "all": "外滩如家"
    }
  }
}



GET /hotel/_search
{
  "query": {
    "multi_match": {
      "query": "外滩如家",
      "fields": ["brand","name","business"]
    }
  }
}




GET /hotel/_search
{
  "query": {
    "term": {
      "city": {
        "value": "上海"
      }
    }
  }
}



GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 200
      }
    }
  }
}



GET /hotel/_search
{
  "query": {
    "geo_distance": {
      "distance": "15km",
      "location": "31.21,121.5"
    }
  }
}




GET /hotel/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "all": "外滩"
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "brand": "如家"
            }
          },
          "weight": 10
        }
      ],
      "boost_mode": "sum"
    }
  }
}





GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "如家"
          }
        }
      ],
      "must_not": [
        {
          "range": {
            "price": {
              "gt": 400
            }
          }
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "10km",
            "location": {
              "lat": 31.21,
              "lon": 121.5
            }
          }
        }
      ]
    }
  }
}




GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "score": "desc" 
    },
    {
      "price": "asc"
    }
  ]
}





GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 31,
          "lon": 121
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}



GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": {
        "order": "asc"
      }
    }
  ],
  "from": 0,
  "size": 20
}



GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": {
        "order": "asc"
      }
    }
  ],
  "from": 9999,
  "size": 20
}


GET /hotel/_search
{
  "query": {
    "match": {
      "all": "如家"
    }
  },
  "highlight": {
    "fields": {
      "name": {
        "require_field_match": "false"
      }
    }
  }
}




POST /hotel/_update/2056126831
{
    "doc": {
        "isAD": true
    }
}
POST /hotel/_update/1989806195
{
    "doc": {
        "isAD": true
    }
}
POST /hotel/_update/2056105938
{
    "doc": {
        "isAD": true
    }
}


GET /hotel/_search
{
  "query": {
    "match": {
      "isAD": "true"
    }
  }
}



GET /hotel/_search
{
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 10
      }
    }
  }
}

GET /hotel/_search
{
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "order": {
          "_count": "asc"
        }, 
        "size": 10
      }
    }
  }
}


GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "lte": 200 
      }
    }
  }, 
  "size": 0, 
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 20
      }
    }
  }
}


GET /hotel/_search
{
  "size": 0, 
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 20
      },
      "aggs": {
        "scoreAgg": {
          "stats": {
            "field": "score"
          }
        }
      }
    }
  }
}


GET /hotel/_search
{
  "size": 0, 
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 20,
        "order": {
          "scoreAgg.avg": "desc"
        }
      },
      "aggs": {
        "scoreAgg": {
          "stats": {
            "field": "score"
          }
        }
      }
    }
  }
}


GET /


POST /_analyze
{
  "text": ["不洗碗工作室"],
  "analyzer": "pinyin"
}

DELETE /test

PUT /test
{
  "settings": {
    "analysis": {
      "analyzer": { 
        "my_analyzer": { 
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": { 
        "py": { 
          "type": "pinyin",
		  "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer",
        "search_analyzer": "ik_smart"
      }
    }
  }
}



POST /test/_doc/1
{
  "id": 1,
  "name": "狮子"
}
POST /test/_doc/2
{
  "id": 2,
  "name": "虱子"
}

GET /test/_search
{
  "query": {
    "match": {
      "name": "掉入狮子笼咋办"
    }
  }
}

// 自动补全的索引库
PUT test2
{
  "mappings": {
    "properties": {
      "title":{
        "type": "completion"
      }
    }
  }
}
// 示例数据
POST test2/_doc
{
  "title": ["Sony", "WH-1000XM3"]
}
POST test2/_doc
{
  "title": ["SK-II", "PITERA"]
}
POST test2/_doc
{
  "title": ["Nintendo", "switch"]
}

POST test2/_search
{
  "query": {
    "match_all": {}
  }
}

# 自动补全查询
POST /test2/_search
{
  "suggest": {
    "title_suggest": {
      "text": "s", 
      "completion": {
        "field": "title", 
        "skip_duplicates": true, 
        "size": 10 
      }
    }
  }
}

GET /hotel/_mapping

DELETE /hotel

# 酒店数据索引库
PUT /hotel
{
  "settings": {
    "analysis": {
      "analyzer": {
        "text_anlyzer": {
          "tokenizer": "ik_max_word",
          "filter": "py"
        },
        "completion_analyzer": {
          "tokenizer": "keyword",
          "filter": "py"
        }
      },
      "filter": {
        "py": {
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id":{
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword",
        "copy_to": "all"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart"
      },
      "suggestion":{
          "type": "completion",
          "analyzer": "completion_analyzer"
      }
    }
  }
}



GET /hotel/_search
{
  "query": {
    "match_all": {}
  }
}


GET /hotel/_search
{
  "suggest": {
    "suggestions": {
      "text":"sd",
      "completion":{
        "field":"suggestion"
      }
    }
  }
}


GET /hotel/_doc/60223





```

