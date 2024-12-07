---
layout: spring Boot
top_img: /images/eye.png
title: Spring Batch处理数据
date: 2024-09-12 15:59:18
categories:
  - Spring Boot
tags:
  - Spring Batch教程
keywords: Spring Boot, Spring Batch
---

## Spring Batch处理数据

2020-03-09 | Visit count

在Spring Batch中，`ItemReader`接口用于读取数据，`ItemWriter`接口用于输出数据。除此之外，我们可以通过`ItemProcessor`接口实现数据的处理，包括：数据校验，数据过滤和数据转换等。数据处理的时机发生于`ItemReader`读取数据之后，`ItemWriter`输出数据之前。本节记录下Spring Batch中`ItemProcessor`的使用。

##  框架搭建

新建一个Spring Boot项目，版本为2.2.4.RELEASE，artifactId为spring-batch-itemprocessor，项目结构如下图所示：
![QQ20200309-135557@2x](https://mrbird.cc/img/QQ20200309-135557@2x.png)

剩下的数据库层的准备，项目配置，依赖引入和[Spring Batch入门](https://mrbird.cc/Spring-Batch%E5%85%A5%E9%97%A8.html)文章中的框架搭建步骤一致，这里就不再赘述。

在介绍Spring Batch ItemProcessor之前，我们先准备个简单的数据读取源。在cc.mrbird.batch包下新建entity包，然后在该包下新建`TestData`实体类：
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
        TestData testData3 = new TestData();  
        testData3.setId(3);  
        testData3.setField1("31");  
        testData3.setField2("32");  
        // 设置为空字符串，用于后面格式校验演示  
        testData3.setField3("");  
        data.add(testData3);  
        return new ListItemReader<>(data);  
    }  
}
```
上面注册了一个`ItemReader`类型的Bean，后续都用它作为读取数据的来源。

## 格式校验
`ItemProcessor`的实现类`ValidatingItemProcessor`可以用于数据格式校验。举个例子，在cc.mrbird.batch包下新建job包，然后在该包下新建`ValidatingItemProcessorDemo`：
```
@Component  
public class ValidatingItemProcessorDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
  
    @Bean  
    public Job validatingItemProcessorJob() {  
        return jobBuilderFactory.get("validatingItemProcessorJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .processor(validatingItemProcessor())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private ValidatingItemProcessor<TestData> validatingItemProcessor() {  
        ValidatingItemProcessor<TestData> processor = new ValidatingItemProcessor<>();  
        processor.setValidator(value -> {  
            // 对每一条数据进行校验  
            if ("".equals(value.getField3())) {  
                // 如果field3的值为空串，则抛异常  
                throw new ValidationException("field3的值不合法");  
            }  
        });  
        return processor;  
    }  
}
```
通过`ValidatingItemProcessor`我们对`ItemReader`读取的每一条数据进行校验，如果field3的值为空串的话，则抛出`ValidationException("field3的值不合法")`异常。`ItemProcessor`通过步骤创建工厂的`processor()`设置。

启动项目，控制台日志的打印如下：
```
1  
2  
3  
4  
5  
6  
7  
8  
9  
10  
11  
12  
13  
14  
15  
16  
17  
18  
19  
20  
21  
22  
23  
24  
25  
26  
27  
28  
29  
30  
31  
32  
33  
34  
35  
36  
37  
38  
39  
40  
41  
42  
43  
44  
45  
46  
47  
48  
49  
50  
51  
52  
53  
54  

2020-03-09 14:18:47.186  INFO 17967 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=validatingItemProcessorJob]] launched with the following parameters: [{}]  
2020-03-09 14:18:47.252  INFO 17967 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
2020-03-09 14:18:47.300 ERROR 17967 --- [           main] o.s.batch.core.step.AbstractStep         : Encountered an error executing step step in job validatingItemProcessorJob  
  
org.springframework.batch.item.validator.ValidationException: field3的值不合法  
	at cc.mrbird.batch.entity.job.ValidatingItemProcessorDemo.lambda$validatingItemProcessor$1(ValidatingItemProcessorDemo.java:50) ~[classes/:na]  
	at org.springframework.batch.item.validator.ValidatingItemProcessor.process(ValidatingItemProcessor.java:84) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.item.SimpleChunkProcessor.doProcess(SimpleChunkProcessor.java:134) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.item.SimpleChunkProcessor.transform(SimpleChunkProcessor.java:319) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.item.SimpleChunkProcessor.process(SimpleChunkProcessor.java:210) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.item.ChunkOrientedTasklet.execute(ChunkOrientedTasklet.java:77) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:407) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:331) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140) ~[spring-tx-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.batch.core.step.tasklet.TaskletStep$2.doInChunkContext(TaskletStep.java:273) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.scope.context.StepContextRepeatCallback.doInIteration(StepContextRepeatCallback.java:82) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.repeat.support.RepeatTemplate.getNextResult(RepeatTemplate.java:375) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.repeat.support.RepeatTemplate.executeInternal(RepeatTemplate.java:215) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.repeat.support.RepeatTemplate.iterate(RepeatTemplate.java:145) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.tasklet.TaskletStep.doExecute(TaskletStep.java:258) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.step.AbstractStep.execute(AbstractStep.java:208) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.job.SimpleStepHandler.handleStep(SimpleStepHandler.java:148) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.job.AbstractJob.handleStep(AbstractJob.java:410) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.job.SimpleJob.doExecute(SimpleJob.java:136) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.job.AbstractJob.execute(AbstractJob.java:319) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.batch.core.launch.support.SimpleJobLauncher$1.run(SimpleJobLauncher.java:147) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.core.task.SyncTaskExecutor.execute(SyncTaskExecutor.java:50) [spring-core-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.batch.core.launch.support.SimpleJobLauncher.run(SimpleJobLauncher.java:140) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_231]  
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_231]  
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_231]  
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_231]  
	at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:344) [spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:198) [spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163) [spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.batch.core.configuration.annotation.SimpleBatchConfiguration$PassthruAdvice.invoke(SimpleBatchConfiguration.java:127) [spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) [spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:212) [spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
	at com.sun.proxy.$Proxy46.run(Unknown Source) [na:na]  
	at org.springframework.boot.autoconfigure.batch.JobLauncherCommandLineRunner.execute(JobLauncherCommandLineRunner.java:192) [spring-boot-autoconfigure-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.autoconfigure.batch.JobLauncherCommandLineRunner.executeLocalJobs(JobLauncherCommandLineRunner.java:166) [spring-boot-autoconfigure-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.autoconfigure.batch.JobLauncherCommandLineRunner.launchJobFromProperties(JobLauncherCommandLineRunner.java:153) [spring-boot-autoconfigure-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.autoconfigure.batch.JobLauncherCommandLineRunner.run(JobLauncherCommandLineRunner.java:148) [spring-boot-autoconfigure-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:784) [spring-boot-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:768) [spring-boot-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:322) [spring-boot-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226) [spring-boot-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1215) [spring-boot-2.2.5.RELEASE.jar:2.2.5.RELEASE]  
	at cc.mrbird.batch.SpringBatchItemprocessorApplication.main(SpringBatchItemprocessorApplication.java:12) [classes/:na]  
  
2020-03-09 14:18:47.307  INFO 17967 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step] executed in 55ms  
2020-03-09 14:18:47.335  INFO 17967 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=validatingItemProcessorJob]] completed with the following parameters: [{}] and the following status: [FAILED] in 127ms
```
可以看到任务处理过程中抛出了预期异常，关于任务处理中如何处理异常，可以参考后续的文章。

除了使用这种方式外，我们还可以使用`BeanValidatingItemProcessor`校验使用JSR-303注解标注的实体类。比如，在TestData类的field3属性上添加`@NotBlank`注解：
```
public class TestData {  
  
    private int id;  
    private String field1;  
    private String field2;  
    @NotBlank  
    private String field3;  
    // get,set,toString略  
}
```
使用该注解需要在pom中添加`spring-boot-starter-validation`依赖：
```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-validation</artifactId>  
</dependency>
```
然后在job包下新建`BeanValidatingItemProcessorDemo`：
```
@Component  
public class BeanValidatingItemProcessorDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
  
    @Bean  
    public Job beanValidatingItemProcessorJob() throws Exception {  
        return jobBuilderFactory.get("beanValidatingItemProcessorJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() throws Exception {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .processor(beanValidatingItemProcessor())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    private BeanValidatingItemProcessor<TestData> beanValidatingItemProcessor() throws Exception {  
        BeanValidatingItemProcessor<TestData> beanValidatingItemProcessor = new BeanValidatingItemProcessor<>();  
        // 开启过滤，不符合规则的数据被过滤掉；  
        beanValidatingItemProcessor.setFilter(true);  
        beanValidatingItemProcessor.afterPropertiesSet();  
        return beanValidatingItemProcessor;  
    }  
}
```
启动项目后，控制台日志打印如下：
```
2020-03-09 14:31:14.813 INFO 18100 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=beanValidatingItemProcessorJob]] launched with the following parameters: [{}]  
2020-03-09 14:31:14.873 INFO 18100 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
2020-03-09 14:31:14.959 INFO 18100 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 85ms  
2020-03-09 14:31:14.980 INFO 18100 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=beanValidatingItemProcessorJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 145ms  
2020-03-09 14:31:15.069 INFO 18100 --- [ main]
```
可以看到，不符合规则的数据已经被排除了。如果不开启过滤`beanValidatingItemProcessor.setFilter(false)`，那么在遇到不符合注解校验规则的数据，将抛出如下异常：
```
org.springframework.batch.item.validator.ValidationException: Validation failed for TestData{id=3, field1='31', field2='32', field3=''}:  
Field error in object 'item' on field 'field3': rejected value []; codes [NotBlank.item.field3,NotBlank.field3,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.field3,field3]; arguments []; default message [field3]]; default message [不能为空]  
at org.springframework.batch.item.validator.SpringValidator.validate(SpringValidator.java:54) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
at org.springframework.batch.item.validator.ValidatingItemProcessor.process(ValidatingItemProcessor.java:84) ~[spring-batch-infrastructure-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
at org.springframework.batch.core.step.item.SimpleChunkProcessor.doProcess(SimpleChunkProcessor.java:134) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
at org.springframework.batch.core.step.item.SimpleChunkProcessor.transform(SimpleChunkProcessor.java:319) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
...
```
## 数据过滤

通过自定义`ItemProcessor`的实现类，我们也可以简单地实现数据过滤。在cc.mrbird.batch包下新建processor包，然后在该包下新建`TestDataFilterItemProcessor`：
```
@Component  
public class TestDataFilterItemProcessor implements ItemProcessor<TestData, TestData> {  
    @Override  
    public TestData process(TestData item) {  
        // 返回null，会过滤掉这条数据  
        return "".equals(item.getField3()) ? null : item;  
    }  
}
```
`TestDataFilterItemProcessor`实现了`ItemProcessor`的`process()`方法，在该方法内编写具体的校验逻辑，上面代码判断TestData的field3是否为空串，是的话返回null（返回null会过滤掉这条数据）。

接着在job包下新建`TestDataFilterItemProcessorDemo`：
```
@Component  
public class TestDataFilterItemProcessorDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private TestDataFilterItemProcessor testDataFilterItemProcessor;  
  
    @Bean  
    public Job testDataFilterItemProcessorJob() {  
        return jobBuilderFactory.get("testDataFilterItemProcessorJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .processor(testDataFilterItemProcessor)  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
}
```
启动项目，控制台日志打印如下：
```
2020-03-09 15:03:30.932  INFO 18690 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=testDataFilterItemProcessorJob]] launched with the following parameters: [{}]  
2020-03-09 15:03:30.973  INFO 18690 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step]  
TestData{id=1, field1='11', field2='12', field3='13'}  
TestData{id=2, field1='21', field2='22', field3='23'}  
2020-03-09 15:03:31.012  INFO 18690 --- [           main] o.s.batch.core.step.AbstractStep         : Step: [step] executed in 39ms  
2020-03-09 15:03:31.037  INFO 18690 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=testDataFilterItemProcessorJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 95ms
```
## 数据转换

在processor包下新建一个`ItemProcessor`实现类`TestDataTransformItemPorcessor`：
```
@Component  
public class TestDataTransformItemPorcessor implements ItemProcessor<TestData, TestData> {  
    @Override  
    public TestData process(TestData item) {  
        // field1值拼接 hello  
        item.setField1(item.getField1() + " hello");  
        return item;  
    }  
}
```
在job包下新建`TestDataTransformItemPorcessorDemo`：
```
@Component  
public class TestDataTransformItemPorcessorDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private TestDataTransformItemPorcessor testDataTransformItemPorcessor;  
  
    @Bean  
    public Job testDataTransformItemPorcessorJob() {  
        return jobBuilderFactory.get("testDataTransformItemPorcessorJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .processor(testDataTransformItemPorcessor)  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
}
```
启动项目，控制台日志打印如下：
```
2020-03-09 15:08:55.628 INFO 18775 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=testDataTransformItemPorcessorJob]] launched with the following parameters: [{}]  
2020-03-09 15:08:55.694 INFO 18775 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11 hello', field2='12', field3='13'}  
TestData{id=2, field1='21 hello', field2='22', field3='23'}  
TestData{id=3, field1='31 hello', field2='32', field3=''}  
2020-03-09 15:08:55.757 INFO 18775 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 63ms  
2020-03-09 15:08:55.781 INFO 18775 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=testDataTransformItemPorcessorJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 144ms
```
## 聚合处理

在创建Step的时候，除了制定一个`ItemProcess`外，我们可以通过`CompositeItemProcessor`聚合多个processor处理过程。

在job包下新建`CompositeItemProcessorDemo`：
```
@Component  
public class CompositeItemProcessorDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private ListItemReader<TestData> simpleReader;  
    @Autowired  
    private TestDataFilterItemProcessor testDataFilterItemProcessor;  
    @Autowired  
    private TestDataTransformItemPorcessor testDataTransformItemPorcessor;  
  
    @Bean  
    public Job compositeItemProcessorJob() {  
        return jobBuilderFactory.get("compositeItemProcessorJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .<TestData, TestData>chunk(2)  
                .reader(simpleReader)  
                .processor(compositeItemProcessor())  
                .writer(list -> list.forEach(System.out::println))  
                .build();  
    }  
  
    // CompositeItemProcessor组合多种中间处理器  
    private CompositeItemProcessor<TestData, TestData> compositeItemProcessor() {  
        CompositeItemProcessor<TestData, TestData> processor = new CompositeItemProcessor<>();  
        List<ItemProcessor<TestData, TestData>> processors = Arrays.asList(testDataFilterItemProcessor, testDataTransformItemPorcessor);  
        // 代理两个processor  
        processor.setDelegates(processors);  
        return processor;  
    }  
}
```
上面代码中，我们通过`CompositeItemProcessor`聚合了前面定义的连个processor：`TestDataFilterItemProcessor`和`TestDataTransformItemPorcessor`。

启动项目，控制台日志打印如下：
```
2020-03-09 15:21:24.960 INFO 18882 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=compositeItemProcessorJob]] launched with the following parameters: [{}]  
2020-03-09 15:21:25.005 INFO 18882 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
TestData{id=1, field1='11 hello', field2='12', field3='13'}  
TestData{id=2, field1='21 hello', field2='22', field3='23'}  
2020-03-09 15:21:25.065 INFO 18882 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 60ms  
2020-03-09 15:21:25.104 INFO 18882 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=compositeItemProcessorJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 128ms
```
从结果可以看到，数据不但进行了过滤，还进行了转换（拼接hello）。


> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[https://github.com/wuyouzhuguli/SpringAll/tree/master/70.spring-batch-itemprocessor](https://github.com/wuyouzhuguli/SpringAll/tree/master/70.spring-batch-itemprocessor)。

