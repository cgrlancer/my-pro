---
layout: spring Boot
top_img: /images/eye.png
title: Spring Batch读取数据
date: 2024-09-12 12:00:12
categories:
  - Spring Boot
tags:
  - Spring Batch教程
keywords: Spring Boot, Spring Batch
---
## Spring Batch读取数据

2020-03-07 | Visit count

Spring Batch读取数据通过ItemReader接口的实现类来完成，包括FlatFileItemReader文本数据读取、StaxEventItemReader XML文件数据读取、JsonItemReader JSON文件数据读取、JdbcPagingItemReader数据库分页数据读取等实现，更多可用的实现可以参考：[https://docs.spring.io/spring-batch/docs/4.2.x/reference/html/appendix.html#itemReadersAppendix](https://docs.spring.io/spring-batch/docs/4.2.x/reference/html/appendix.html#itemReadersAppendix)，本文只介绍这四种比较常用的读取数据方式。

## [](https://mrbird.cc/Spring-Batch%E8%AF%BB%E5%8F%96%E6%95%B0%E6%8D%AE.html#%E6%A1%86%E6%9E%B6%E6%90%AD%E5%BB%BA "框架搭建")框架搭建

新建一个Spring Boot项目，版本为2.2.4.RELEASE，artifactId为spring-batch-itemreader，项目结构如下图所示：
![QQ20200307-104004@2x](https://mrbird.cc/img/QQ20200307-104004@2x.png)
剩下的数据库层的准备，项目配置，依赖引入和[Spring Batch入门](https://mrbird.cc/Spring-Batch%E5%85%A5%E9%97%A8.html)文章中的框架搭建步骤一致，这里就不再赘述。

## [](https://mrbird.cc/Spring-Batch%E8%AF%BB%E5%8F%96%E6%95%B0%E6%8D%AE.html#%E7%AE%80%E5%8D%95%E6%95%B0%E6%8D%AE%E8%AF%BB%E5%8F%96 "简单数据读取")简单数据读取

前面提到，Spring Batch读取数据是通过ItemReader接口的实现类来完成的，所以我们可以自定义一个ItemReader的实现类，实现简单数据的读取。

在cc.mrbird.batch包下新建reader包，然后在该包下新建ItemReader接口的实现类`MySimpleIteamReader`：
```
public class MySimpleIteamReader implements ItemReader<String> {  
  
    private Iterator<String> iterator;  
  
    public MySimpleIteamReader(List<String> data) {  
        this.iterator = data.iterator();  
    }  
  
    @Override  
    public String read() {  
        // 数据一个接着一个读取  
        return iterator.hasNext() ? iterator.next() : null;  
    }  
}
```
泛型指定读取数据的格式，这里读取的是String类型的List，`read()`方法的实现也很简单，就是遍历集合数据。

接着在cc.mrbird.batch包下新建job包，然后在该包下新建`MySimpleItemReaderDemo`类，用于测试我们定义的`MySimpleIteamReader`，`MySimpleItemReaderDemo`类代码如下：
```
@Component  
public class MySimpleItemReaderDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job mySimpleItemReaderJob() {  
        return jobBuilderFactory.get("mySimpleItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<String, String>chunk(2)  
                .reader(mySimpleItemReader())  
                .writer(list -> list.forEach(System.out::println))  // 简单输出，后面再详细介绍writer  
                .build();  
    }  
  
    private ItemReader<String> mySimpleItemReader() {  
        List<String> data = Arrays.asList("java", "c++", "javascript", "python");  
        return new MySimpleIteamReader(data);  
    }  
}
```
上面代码中，我们通过`mySimpleItemReader()`方法创建了一个`MySimpleIteamReader`，并且传入了List数据。上面代码大体和上一节中介绍的差不多，最主要的区别就是Step的创建过程稍有不同。

在`MySimpleItemReaderDemo`类中，我们通过`StepBuilderFactory`创建步骤Step，不过不再是使用`tasklet()`方法创建，而是使用`chunk()`方法。chunk字面上的意思是“块”的意思，可以简单理解为数据块，泛型`<String, String>`用于指定读取的数据和输出的数据类型，构造器入参指定了数据块的大小，比如指定为2时表示每当读取2组数据后做一次数据输出处理。接着`reader()`方法指定读取数据的方式，该方法接收`ItemReader`的实现类，这里使用的是我们自定义的`MySimpleIteamReader`。`writer()`方法指定数据输出方式，因为这块不是本文的重点，所以先简单遍历输出即可。

启动项目，控制台日志打印如下：
```
2020-03-07 11:17:32.303 INFO 28381 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=mySimpleItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 11:17:32.369 INFO 28381 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
java  
c++  
javascript  
python  
2020-03-07 11:17:32.428 INFO 28381 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 59ms  
2020-03-07 11:17:32.451 INFO 28381 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=mySimpleItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 125ms
```
## 文本数据读取

Spring Batch读取文本类型数据可以通过`FlatFileItemReader`实现，在演示怎么使用之前，我们先准备好数据文件。

在resources目录下新建file文件，内容如下：
```
// 演示文件数据读取  
1,11,12,13  
2,21,22,23  
3,31,32,33  
4,41,42,43  
5,51,52,53  
6,61,62,63
```
file的数据是一行一行以逗号分隔的数据（在批处理业务中，文本类型的数据文件一般都是有一定规律的）。在文本数据读取的过程中，我们需要将读取的数据转换为POJO对象存储，所以我们需要创建一个与之对应的POJO对象。在cc.mrbird.batch包下新建entity包，然后在该包下新建`TestData`类：
```
public class TestData {  
    private int id;  
    private String field1;  
    private String field2;  
    private String field3;  
  
    // get,set,toString略  
}
```
因为file文本中的一行数据经过逗号分隔后为1、11、12、13，所以我们创建的与之对应的POJO TestData包含4个属性id、field1、field2和field3。

接着在job包下新建`FileItemReaderDemo`：
```
@Component  
public class FileItemReaderDemo {  
    // 任务创建工厂  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    // 步骤创建工厂  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job fileItemReaderJob() {  
        return jobBuilderFactory.get("fileItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(fileItemReader())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ItemReader<TestData> fileItemReader() {  
        FlatFileItemReader<TestData> reader = new FlatFileItemReader<>();  
        reader.setResource(new ClassPathResource("file")); // 设置文件资源地址  
        reader.setLinesToSkip(1); // 忽略第一行  
  
        // AbstractLineTokenizer的三个实现类之一，以固定分隔符处理行数据读取,  
        // 使用默认构造器的时候，使用逗号作为分隔符，也可以通过有参构造器来指定分隔符  
        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();  
  
        // 设置属性名，类似于表头  
        tokenizer.setNames("id", "field1", "field2", "field3");  
  
        // 将每行数据转换为TestData对象  
        DefaultLineMapper<TestData> mapper = new DefaultLineMapper<>();  
        // 设置LineTokenizer  
        mapper.setLineTokenizer(tokenizer);  
  
        // 设置映射方式，即读取到的文本怎么转换为对应的POJO  
        mapper.setFieldSetMapper(fieldSet -> {  
            TestData data = new TestData();  
            data.setId(fieldSet.readInt("id"));  
            data.setField1(fieldSet.readString("field1"));  
            data.setField2(fieldSet.readString("field2"));  
            data.setField3(fieldSet.readString("field3"));  
            return data;  
        });  
        reader.setLineMapper(mapper);  
        return reader;  
    }  
}
```
上面代码中，我们在`fileItemReader()`方法里编写了具体的文本数据读取代码，过程参考注释即可。`DelimitedLineTokenizer`分隔符行处理器的默认构造器源码如下所示：
![QQ20200307-115347@2x](https://mrbird.cc/img/QQ20200307-115347@2x.png)
常量`DELIMITER_COMMA`的值为`public static final String DELIMITER_COMMA = ",";`，假如我们的数据并不是用逗号分隔，而是用`|`等字符分隔的话，可以使用它的有参构造器指定：

> DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer("|");

`DelimitedLineTokenizer`是`AbstractLineTokenizer`三个实现类之一：
![QQ20200307-115730@2x](https://mrbird.cc/img/QQ20200307-115730@2x.png)
顾名思义，`FixedLengthTokenizer`通过指定的固定长度来截取数据，`RegexLineTokenizer`通过正则表达式来匹配数据，这里就不演示了，有兴趣的可以自己玩玩。

编写好`FileItemReaderDemo`后，启动项目，控制台日志打印如下：
```
2020-03-07 12:06:11.876 INFO 29042 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=fileItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 12:06:11.937 INFO 29042 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
TestData{id=3, field1='31', field2='32', field3='33'}  
TestData{id=4, field1='41', field2='42', field3='43'}  
TestData{id=5, field1='51', field2='52', field3='53'}  
TestData{id=6, field1='61', field2='62', field3='63'}  
2020-03-07 12:06:12.020 INFO 29042 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 83ms  
2020-03-07 12:06:12.044 INFO 29042 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=fileItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 146ms
```
## 数据库数据读取

在演示从数据库中读取数据之前，我们先准备好测试数据。在springbatch数据库中新建一张TEST表，SQL语句如下所示：
```
-- ----------------------------  
-- Table structure for TEST  
-- ----------------------------  
DROP TABLE IF EXISTS `TEST`;  
CREATE TABLE `TEST` (  
  `id` bigint(10) NOT NULL COMMENT 'ID',  
  `field1` varchar(10) NOT NULL COMMENT '字段一',  
  `field2` varchar(10) NOT NULL COMMENT '字段二',  
  `field3` varchar(10) NOT NULL COMMENT '字段三',  
  PRIMARY KEY (`id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
  
-- ----------------------------  
-- Records of TEST  
-- ----------------------------  
BEGIN;  
INSERT INTO `TEST` VALUES (1, '11', '12', '13');  
INSERT INTO `TEST` VALUES (2, '21', '22', '23');  
INSERT INTO `TEST` VALUES (3, '31', '32', '33');  
INSERT INTO `TEST` VALUES (4, '41', '42', '43');  
INSERT INTO `TEST` VALUES (5, '51', '52', '53');  
INSERT INTO `TEST` VALUES (6, '61', '62', '63');  
COMMIT;
```
TEST表的字段和上面创建的`TestData`实体类一致。

然后在job包下新建`DataSourceItemReaderDemo`类，测试从数据库中读取数据：
```
@Component  
public class DataSourceItemReaderDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    // 注入数据源  
    @Autowired  
    private DataSource dataSource;  
  
    @Bean  
    public Job dataSourceItemReaderJob() throws Exception {  
        return jobBuilderFactory.get("dataSourceItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() throws Exception {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(dataSourceItemReader())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ItemReader<TestData> dataSourceItemReader() throws Exception {  
        JdbcPagingItemReader<TestData> reader = new JdbcPagingItemReader<>();  
        reader.setDataSource(dataSource); // 设置数据源  
        reader.setFetchSize(5); // 每次取多少条记录  
        reader.setPageSize(5); // 设置每页数据量  
  
        // 指定sql查询语句 select id,field1,field2,field3 from TEST  
        MySqlPagingQueryProvider provider = new MySqlPagingQueryProvider();  
        provider.setSelectClause("id,field1,field2,field3"); //设置查询字段  
        provider.setFromClause("from TEST"); // 设置从哪张表查询  
  
        // 将读取到的数据转换为TestData对象  
        reader.setRowMapper((resultSet, rowNum) -> {  
            TestData data = new TestData();  
            data.setId(resultSet.getInt(1));  
            data.setField1(resultSet.getString(2)); // 读取第一个字段，类型为String  
            data.setField2(resultSet.getString(3));  
            data.setField3(resultSet.getString(4));  
            return data;  
        });  
  
        Map<String, Order> sort = new HashMap<>(1);  
        sort.put("id", Order.ASCENDING);  
        provider.setSortKeys(sort); // 设置排序,通过id 升序  
  
        reader.setQueryProvider(provider);  
  
        // 设置namedParameterJdbcTemplate等属性  
        reader.afterPropertiesSet();  
        return reader;  
    }  
}
```

`dataSourceItemReader()`方法中的主要步骤就是：通过`JdbcPagingItemReader`设置对应的数据源，然后设置数据量、获取数据的sql语句、排序规则和查询结果与POJO的映射规则等。方法末尾之所以需要调用`JdbcPagingItemReader`的`afterPropertiesSet()`方法是因为需要设置JDBC模板（`afterPropertiesSet()`方法源码）：
![QQ20200307-155834@2x](https://mrbird.cc/img/QQ20200307-155834@2x.png)
启动项目，控制台日志打印如下：
```
2020-03-07 16:01:05.366 INFO 30264 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=dataSourceItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 16:01:05.420 INFO 30264 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
TestData{id=3, field1='31', field2='32', field3='33'}  
TestData{id=4, field1='41', field2='42', field3='43'}  
TestData{id=5, field1='51', field2='52', field3='53'}  
TestData{id=6, field1='61', field2='62', field3='63'}  
2020-03-07 16:01:05.512 INFO 30264 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 92ms  
2020-03-07 16:01:05.534 INFO 30264 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=dataSourceItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 147ms
```
## XML数据读取

Spring Batch借助Spring OXM可以轻松地实现xml格式数据文件读取。在resources目录下新建file.xml，内容如下所示：
```
<?xml version="1.0" encoding="utf-8" ?>  
<tests>  
    <test>  
        <id>1</id>  
        <field1>11</field1>  
        <field2>12</field2>  
        <field3>13</field3>  
    </test>  
    <test>  
        <id>2</id>  
        <field1>21</field1>  
        <field2>22</field2>  
        <field3>23</field3>  
    </test>  
    <test>  
        <id>3</id>  
        <field1>31</field1>  
        <field2>32</field2>  
        <field3>33</field3>  
    </test>  
    <test>  
        <id>4</id>  
        <field1>41</field1>  
        <field2>42</field2>  
        <field3>43</field3>  
    </test>  
    <test>  
        <id>5</id>  
        <field1>51</field1>  
        <field2>52</field2>  
        <field3>53</field3>  
    </test>  
    <test>  
        <id>6</id>  
        <field1>61</field1>  
        <field2>62</field2>  
        <field3>63</field3>  
    </test>  
</tests>
```
xml文件内容由一组一组的`<test></test>`标签组成，`<test>`标签又包含四组子标签，标签名称和`TestData`实体类属性一一对应。

准备好xml文件后，我们在pom中引入spring-oxm依赖：
```
<dependency>  
    <groupId>org.springframework</groupId>  
    <artifactId>spring-oxm</artifactId>  
</dependency>  
<dependency>  
    <groupId>com.thoughtworks.xstream</groupId>  
    <artifactId>xstream</artifactId>  
    <version>1.4.11.1</version>  
</dependency>
```
接着在job包下新建`XmlFileItemReaderDemo`，演示xml文件数据获取：
```
@Component  
public class XmlFileItemReaderDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job xmlFileItemReaderJob() {  
        return jobBuilderFactory.get("xmlFileItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(xmlFileItemReader())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ItemReader<TestData> xmlFileItemReader() {  
        StaxEventItemReader<TestData> reader = new StaxEventItemReader<>();  
        reader.setResource(new ClassPathResource("file.xml")); // 设置xml文件源  
        reader.setFragmentRootElementName("test"); // 指定xml文件的根标签  
        // 将xml数据转换为TestData对象  
        XStreamMarshaller marshaller = new XStreamMarshaller();  
        // 指定需要转换的目标数据类型  
        Map<String, Class<TestData>> map = new HashMap<>(1);  
        map.put("test", TestData.class);  
        marshaller.setAliases(map);  
  
        reader.setUnmarshaller(marshaller);  
        return reader;  
    }  
}
```
在`xmlFileItemReader()`方法中，我们通过`StaxEventItemReader`读取xml文件，代码较简单，看注释即可。

启动项目，控制台日志打印如下：
```
020-03-07 16:23:47.775 INFO 30450 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=xmlFileItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 16:23:47.820 INFO 30450 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
TestData{id=3, field1='31', field2='32', field3='33'}  
TestData{id=4, field1='41', field2='42', field3='43'}  
TestData{id=5, field1='51', field2='52', field3='53'}  
TestData{id=6, field1='61', field2='62', field3='63'}  
2020-03-07 16:23:47.961 INFO 30450 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 140ms  
2020-03-07 16:23:47.984 INFO 30450 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=xmlFileItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 200ms
```
## JSON数据读取

在resources目录下新建file.json文件，内容如下：
```
[  
  {  
    "id": 1,  
    "field1": "11",  
    "field2": "12",  
    "field3": "13"  
  },  
  {  
    "id": 2,  
    "field1": "21",  
    "field2": "22",  
    "field3": "23"  
  },  
  {  
    "id": 3,  
    "field1": "31",  
    "field2": "32",  
    "field3": "33"  
  }  
]
```
JSON对象属性和TestData对象属性一一对应。在job包下新建`JSONFileItemReaderDemo`，用于测试JSON文件数据读取：
```
@Component  
public class JSONFileItemReaderDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job jsonFileItemReaderJob() {  
        return jobBuilderFactory.get("jsonFileItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(jsonItemReader())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ItemReader<TestData> jsonItemReader() {  
        // 设置json文件地址  
        ClassPathResource resource = new ClassPathResource("file.json");  
        // 设置json文件转换的目标对象类型  
        JacksonJsonObjectReader<TestData> jacksonJsonObjectReader = new JacksonJsonObjectReader<>(TestData.class);  
        JsonItemReader<TestData> reader = new JsonItemReader<>(resource, jacksonJsonObjectReader);  
        // 给reader设置一个别名  
        reader.setName("testDataJsonItemReader");  
        return reader;  
    }  
}
```
启动项目，控制台输出如下：
```
2020-03-07 16:40:52.508 INFO 30599 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=jsonFileItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 16:40:52.554 INFO 30599 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
TestData{id=3, field1='31', field2='32', field3='33'}  
2020-03-07 16:40:52.622 INFO 30599 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 67ms  
2020-03-07 16:40:52.642 INFO 30599 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=jsonFileItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 124ms
```
## 多文本数据读取

多文本的数据读取本质还是单文件数据读取，区别就是多文件读取需要在单文件读取的方式上设置一层代理。

在resources目录下新建两个文件file1和file2，file1内容如下所示：

>// 演示文件数据读取  
1,11,12,13  
2,21,22,23  
3,31,32,33  
4,41,42,43  
5,51,52,53  
6,61,62,63

file2内容如下所示：

> // 演示文件数据读取  
7,71,72,73  
8,81,82,83

然后在job包下新建`MultiFileIteamReaderDemo`，演示多文件数据读取：
```
@Component  
public class MultiFileIteamReaderDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
  
    @Bean  
    public Job multiFileItemReaderJob() {  
        return jobBuilderFactory.get("multiFileItemReaderJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(multiFileItemReader())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ItemReader<TestData> multiFileItemReader() {  
        MultiResourceItemReader<TestData> reader = new MultiResourceItemReader<>();  
        reader.setDelegate(fileItemReader()); // 设置文件读取代理，方法可以使用前面文件读取中的例子  
  
        Resource[] resources = new Resource[]{  
                new ClassPathResource("file1"),  
                new ClassPathResource("file2")  
        };  
  
        reader.setResources(resources); // 设置多文件源  
        return reader;  
    }  
  
    private FlatFileItemReader<TestData> fileItemReader() {  
        FlatFileItemReader<TestData> reader = new FlatFileItemReader<>();  
        reader.setLinesToSkip(1); // 忽略第一行  
  
        // AbstractLineTokenizer的三个实现类之一，以固定分隔符处理行数据读取,  
        // 使用默认构造器的时候，使用逗号作为分隔符，也可以通过有参构造器来指定分隔符  
        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();  
        // 设置属姓名，类似于表头  
        tokenizer.setNames("id", "field1", "field2", "field3");  
        // 将每行数据转换为TestData对象  
        DefaultLineMapper<TestData> mapper = new DefaultLineMapper<>();  
        mapper.setLineTokenizer(tokenizer);  
        // 设置映射方式  
        mapper.setFieldSetMapper(fieldSet -> {  
            TestData data = new TestData();  
            data.setId(fieldSet.readInt("id"));  
            data.setField1(fieldSet.readString("field1"));  
            data.setField2(fieldSet.readString("field2"));  
            data.setField3(fieldSet.readString("field3"));  
            return data;  
        });  
  
        reader.setLineMapper(mapper);  
        return reader;  
    }  
}
```
上面代码中`fileItemReader()`方法在**文本数据读取**中介绍过了，多文件读取的关键在于`multiFileItemReader()`方法，该方法通过`MultiResourceItemReader`对象设置了多个文件的目标地址，并且将单文件的读取方式设置为代理。

启动项目，控制台日志打印如下：
```
2020-03-07 16:55:24.480  INFO 30749 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=multiFileItemReaderJob]] launched with the following parameters: [{}]  
2020-03-07 16:55:24.536  INFO 30749 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
TestData{id=3, field1='31', field2='32', field3='33'}  
TestData{id=4, field1='41', field2='42', field3='43'}  
TestData{id=5, field1='51', field2='52', field3='53'}  
TestData{id=6, field1='61', field2='62', field3='63'}  
TestData{id=7, field1='71', field2='72', field3='73'}  
TestData{id=8, field1='81', field2='82', field3='83'}  
2020-03-07 16:55:24.617  INFO 30749 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step] executed in 81ms  
2020-03-07 16:55:24.643  INFO 30749 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=multiFileItemReaderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 153ms
```
> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[https://github.com/wuyouzhuguli/SpringAll/tree/master/68.spring-batch-itemreader](https://github.com/wuyouzhuguli/SpringAll/tree/master/68.spring-batch-itemreader)。
