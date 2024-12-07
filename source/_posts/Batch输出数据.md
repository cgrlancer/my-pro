---
layout: spring Boot
top_img: /images/eye.png
title: Spring Batch输出数据
date: 2024-09-12 12:32:25
categories:
  - Spring Boot
tags:
  - Spring Batch教程
keywords: Spring Boot, Spring Batch  
---
Spring Batch输出数据通过ItemWriter接口的实现类来完成，包括FlatFileItemWriter文本数据输出、StaxEventItemWriter XML文件数据输出、JsonItemWriter JSON文件数据输出、JdbcBatchItemWriter数据库数据插入等实现，更多可用的实现可以参考：[https://docs.spring.io/spring-batch/docs/4.2.x/reference/html/appendix.html#itemWritersAppendix](https://docs.spring.io/spring-batch/docs/4.2.x/reference/html/appendix.html#itemWritersAppendix)，本文只介绍这四种比较常用的输出数据方式。
## 框架搭建

新建一个Spring Boot项目，版本为2.2.4.RELEASE，artifactId为spring-batch-itemwriter，项目结构如下图所示：
![QQ20200309-092102@2x](https://mrbird.cc/img/QQ20200309-092102@2x.png)
剩下的数据库层的准备，项目配置，依赖引入和[Spring Batch入门](https://mrbird.cc/Spring-Batch%E5%85%A5%E9%97%A8.html)文章中的框架搭建步骤一致，这里就不再赘述。

在介绍Spring Batch数据输出之前，我们先准备个简单的数据读取源。在cc.mrbird.batch包下新建entity包，然后在该包下新建`TestData`实体类：
```
public class TestData {  
  
    private int id;  
    private String field1;  
    private String field2;  
    private String field3;  
    // get,set,toString略  
}
```
接着在cc.mrbird.batch包下新建reader包，然后在该包下创建`ItemReaderConfigure`：
```
@Configuration  
public class ItemReaderConfigure {  
  
    @Bean  
    public ListItemReader<TestData> simpleReader() {  
        List<TestData> data = new ArrayList<>();  
        TestData testData1 = new TestData();  
        testData1.setId(1);  
        testData1.setField1("11");  
        testData1.setField2("12");  
        testData1.setField3("13");  
        data.add(testData1);  
        TestData testData2 = new TestData();  
        testData2.setId(2);  
        testData2.setField1("21");  
        testData2.setField2("22");  
        testData2.setField3("23");  
        data.add(testData2);  
        return new ListItemReader<>(data);  
    }  
}
```
上面注册了一个ItemReader类型的Bean，后续都用它作为读取数据的来源。
## 输出文本数据

在cc.mrbird.batch包下新建job包，然后在该包下新建`FileItemWriterDemo`，用于测试Spring Batch输出数据到文本文件：
```
@Component  
public class FileItemWriterDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
  
    @Bean  
    public Job fileItemWriterJob() throws Exception {  
        return jobBuilderFactory.get("fileItemWriterJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() throws Exception {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(fileItemWriter())  
                .build();  
    }  
  
    private FlatFileItemWriter<TestData> fileItemWriter() throws Exception {  
        FlatFileItemWriter<TestData> writer = new FlatFileItemWriter<>();  
  
        FileSystemResource file = new FileSystemResource("/Users/mrbird/Desktop/file");  
        Path path = Paths.get(file.getPath());  
        if (!Files.exists(path)) {  
            Files.createFile(path);  
        }  
        // 设置输出文件路径  
        writer.setResource(file);   
  
        // 把读到的每个TestData对象转换为JSON字符串  
        LineAggregator<TestData> aggregator = item -> {  
            try {  
                ObjectMapper mapper = new ObjectMapper();  
                return mapper.writeValueAsString(item);  
            } catch (JsonProcessingException e) {  
                e.printStackTrace();  
            }  
            return "";  
        };  
  
        writer.setLineAggregator(aggregator);  
        writer.afterPropertiesSet();  
        return writer;  
    }  
}
```
上面代码中，Step中的Reader使用的是我们上面创建的`simpleReader`，文本数据输出使用的是`FlatFileItemWriter`。`fileItemWriter()`方法的代码较为简单，这里就不赘述了。

启动项目后，在`/Users/mrbird/Desktop`目录下（也就是我的电脑桌面上）会多出个file文件：
![QQ20200309-094315@2x](https://mrbird.cc/img/QQ20200309-094315@2x.png)

## 输出xml数据

同样的，xml格式数据输出需要借助spring-oxm框架，在pom中引入相关依赖：
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
然后在job包下新建`XmlFileItemWriterDemo`，用于测试Spring Batch输出数据到xml文件：
```
@Component  
public class XmlFileItemWriterDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
  
    @Bean  
    public Job xmlFileItemWriterJob() throws Exception {  
        return jobBuilderFactory.get("xmlFileItemWriterJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() throws Exception {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(xmlFileItemWriter())  
                .build();  
    }  
  
    private StaxEventItemWriter<TestData> xmlFileItemWriter() throws IOException {  
        StaxEventItemWriter<TestData> writer = new StaxEventItemWriter<>();  
  
        // 通过XStreamMarshaller将TestData转换为xml  
        XStreamMarshaller marshaller = new XStreamMarshaller();  
  
        Map<String,Class<TestData>> map = new HashMap<>(1);  
        map.put("test", TestData.class);  
  
        marshaller.setAliases(map); // 设置xml标签  
  
        writer.setRootTagName("tests"); // 设置根标签  
        writer.setMarshaller(marshaller);  
  
        FileSystemResource file = new FileSystemResource("/Users/mrbird/Desktop/file.xml");  
        Path path = Paths.get(file.getPath());  
        if (!Files.exists(path)) {  
            Files.createFile(path);  
        }  
  
        writer.setResource(file); // 设置目标文件路径  
        return writer;  
    }  
}
```
xml类型文件输出使用的是`StaxEventItemWriter`。

启动项目后，在`/Users/mrbird/Desktop`目录下会多出个file.xml文件：
![QQ20200309-095332@2x](https://mrbird.cc/img/QQ20200309-095332@2x.png)

## 输出JSON数据

在job包下新建`JsonFileItemWriterDemo`，用于测试Spring Batch输出数据到json文件：
```
@Component  
public class JsonFileItemWriterDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
  
    @Bean  
    public Job jsonFileItemWriterJob() throws Exception {  
        return jobBuilderFactory.get("jsonFileItemWriterJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() throws Exception {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(jsonFileItemWriter())  
                .build();  
    }  
  
    private JsonFileItemWriter<TestData> jsonFileItemWriter() throws IOException {  
        // 文件输出目标地址  
        FileSystemResource file = new FileSystemResource("/Users/mrbird/Desktop/file.json");  
        Path path = Paths.get(file.getPath());  
        if (!Files.exists(path)) {  
            Files.createFile(path);  
        }  
        // 将对象转换为json  
        JacksonJsonObjectMarshaller<TestData> marshaller = new JacksonJsonObjectMarshaller<>();  
        JsonFileItemWriter<TestData> writer = new JsonFileItemWriter<>(file, marshaller);  
        // 设置别名  
        writer.setName("testDatasonFileItemWriter");  
        return writer;  
    }  
}
```
json类型文件输出使用的是`JsonFileItemWriter`。

启动项目后，在`/Users/mrbird/Desktop`目录下会多出个file.json文件：
![QQ20200309-100359@2x](https://mrbird.cc/img/QQ20200309-100359@2x.png)

## 输出数据到数据库

在job包下新建`DatabaseItemWriterDemo`，用于测试Spring Batch输出数据到数据库：
```
@Component  
public class DatabaseItemWriterDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private DataSource dataSource;  
  
    @Bean  
    public Job datasourceItemWriterJob() {  
        return jobBuilderFactory.get("datasourceItemWriterJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(dataSourceItemWriter())  
                .build();  
    }  
  
    private ItemWriter<TestData> dataSourceItemWriter() {  
        // ItemWriter的实现类之一，mysql数据库数据写入使用JdbcBatchItemWriter，  
        // 其他实现：MongoItemWriter,Neo4jItemWriter等  
        JdbcBatchItemWriter<TestData> writer = new JdbcBatchItemWriter<>();  
        writer.setDataSource(dataSource); // 设置数据源  
  
        String sql = "insert into TEST(id,field1,field2,field3) values (:id,:field1,:field2,:field3)";  
        writer.setSql(sql); // 设置插入sql脚本  
  
        // 映射TestData对象属性到占位符中的属性  
        BeanPropertyItemSqlParameterSourceProvider<TestData> provider = new BeanPropertyItemSqlParameterSourceProvider<>();  
        writer.setItemSqlParameterSourceProvider(provider);  
  
        writer.afterPropertiesSet(); // 设置一些额外属性  
        return writer;  
    }  
}
```
MySQL关系型数据数据写入使用的是`JdbcBatchItemWriter`。在测试之前，先清空springbatch数据库TEST表数据，然后启动项目，启动后，TEST表记录如下所示：
![QQ20200309-102006@2x](https://mrbird.cc/img/QQ20200309-102006@2x.png)

## 多文本输出

多文本输出和上一节介绍的多文本数据读取类似，都是需要通过代理来完成。我们模拟个同时输出xml格式和普通文本格式的例子。

在cc.mrbird.batch包下新建writer包，然后在该包下新建`ItemWriterConfigure`配置类：
```
@Configuration  
public class ItemWriterConfigure {  
  
    @Bean  
    public FlatFileItemWriter<TestData> fileItemWriter() throws Exception {  
        FlatFileItemWriter<TestData> writer = new FlatFileItemWriter<>();  
  
        FileSystemResource file = new FileSystemResource("/Users/mrbird/Desktop/file");  
        Path path = Paths.get(file.getPath());  
        if (!Files.exists(path)) {  
            Files.createFile(path);  
        }  
  
        writer.setResource(file); // 设置目标文件路径  
  
        // 把读到的每个TestData对象转换为字符串  
        LineAggregator<TestData> aggregator = item -> {  
            try {  
                ObjectMapper mapper = new ObjectMapper();  
                return mapper.writeValueAsString(item);  
            } catch (JsonProcessingException e) {  
                e.printStackTrace();  
            }  
            return "";  
        };  
  
        writer.setLineAggregator(aggregator);  
        writer.afterPropertiesSet();  
        return writer;  
    }  
  
    @Bean  
    public StaxEventItemWriter<TestData> xmlFileItemWriter() throws Exception {  
        StaxEventItemWriter<TestData> writer = new StaxEventItemWriter<>();  
  
        // 通过XStreamMarshaller将TestData转换为xml  
        XStreamMarshaller marshaller = new XStreamMarshaller();  
  
        Map<String, Class<TestData>> map = new HashMap<>(1);  
        map.put("test", TestData.class);  
  
        marshaller.setAliases(map); // 设置xml标签  
  
        writer.setRootTagName("tests"); // 设置根标签  
        writer.setMarshaller(marshaller);  
  
        FileSystemResource file = new FileSystemResource("/Users/mrbird/Desktop/file.xml");  
        Path path = Paths.get(file.getPath());  
        if (!Files.exists(path)) {  
            Files.createFile(path);  
        }  
  
        writer.setResource(file); // 设置目标文件路径  
        return writer;  
    }  
}
```
上面的配置类中，配置了`FlatFileItemWriter`和`StaxEventItemWriter`类型的ItemWriter Bean，代码步骤和前面介绍的一致。

然后在job包下新建`MultiFileItemWriteDemo`，用于测试多文本输出：
```
@Component  
public class MultiFileItemWriteDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private DataSource dataSource;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private ItemStreamWriter<TestData> fileItemWriter;  
    @Autowired  
    private ItemStreamWriter<TestData> xmlFileItemWriter;  
  
    @Bean  
    public Job multiFileItemWriterJob() {  
        return jobBuilderFactory.get("multiFileItemWriterJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(classifierMultiFileItemWriter())  
                .stream(fileItemWriter)  
                .stream(xmlFileItemWriter)  
                .build();  
    }  
  
    // 将数据分类，然后分别输出到对应的文件(此时需要将writer注册到ioc容器，否则报  
    // WriterNotOpenException: Writer must be open before it can be written to)  
    private ClassifierCompositeItemWriter<TestData> classifierMultiFileItemWriter() {  
        ClassifierCompositeItemWriter<TestData> writer = new ClassifierCompositeItemWriter<>();  
        writer.setClassifier((Classifier<TestData, ItemWriter<? super TestData>>) testData -> {  
            try {  
                // id能被2整除则输出到普通文本，否则输出到xml文本  
                return testData.getId() % 2 == 0 ? fileItemWriter : xmlFileItemWriter;  
            } catch (Exception e) {  
                e.printStackTrace();  
            }  
            return null;  
        });  
        return writer;  
    }  
}
```
`ClassifierCompositeItemWriter`可以设置不同条件下使用不同的ItemWriter输出数据，此外在Step中，还需通过`StepBuilderFactory`的`stream()`方法传入使用到的ItemWriter（这里需要注意的是，注入的时候，类型应选择ItemStreamWriter）。

在启动项目前，先删掉`/Users/mrbird/Desktop`目录下的文件。删掉后，启动项目，结果如下：
![QQ20200309-110058@2x](https://mrbird.cc/img/QQ20200309-110058@2x.png)
![QQ20200309-110519@2x](https://mrbird.cc/img/QQ20200309-110519@2x.png)

如果不想用分类，希望所有数据都输出到对应格式的文本中，则可以使用`CompositeItemWriter`作为代理输出，修改`MultiFileItemWriteDemo`：
```
@Component  
public class MultiFileItemWriteDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private ItemStreamWriter<TestData> fileItemWriter;  
    @Autowired  
    private ItemStreamWriter<TestData> xmlFileItemWriter;  
  
    @Bean  
    public Job multiFileItemWriterJob() {  
        return jobBuilderFactory.get("multiFileItemWriterJob2")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .writer(multiFileItemWriter())  
                .build();  
    }  
  
    // 输出数据到多个文件  
    private CompositeItemWriter<TestData> multiFileItemWriter() {  
        // 使用CompositeItemWriter代理  
        CompositeItemWriter<TestData> writer = new CompositeItemWriter<>();  
        // 设置具体写代理  
        writer.setDelegates(Arrays.asList(fileItemWriter, xmlFileItemWriter));  
        return writer;  
    }  
}
```
在启动项目前，先删掉`/Users/mrbird/Desktop`目录下的文件。删掉后，启动项目，结果如下：
![QQ20200309-111155@2x](https://mrbird.cc/img/QQ20200309-111155@2x.png)

> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[https://github.com/wuyouzhuguli/SpringAll/tree/master/69.spring-batch-itemwriter](https://github.com/wuyouzhuguli/SpringAll/tree/master/69.spring-batch-itemwriter)。
