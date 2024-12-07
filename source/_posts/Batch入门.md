---
layout: spring Boot
top_img: /images/eye.png
title: Spring Batch入门
date: 2024-09-12 11:13:59
ategories:
  - Spring Boot
tags:
  - Spring Batch教程
keywords: Spring Boot, Spring Batch

---

## Spring Batch入门
企业中经常会有需要批处理才能完成的业务操作，比如：自动化地处理大批量复杂的数据，如月结计算；重复性地处理大批量数据，如费率计算；充当内部系统和外部系统的数据纽带，中间需要对数据进行格式化，校验，转换处理等。

Spring Batch是一个轻量级但功能又十分全面的批处理框架，本节我们将通过一些简单的例子来入门Spring Batch。
## 框架搭建
新建一个Spring Boot项目，版本为2.2.4.RELEASE，artifactId为spring-batch-start，项目结构如下图所示：
![QQ20200306-095608@2x](https://mrbird.cc/img/QQ20200306-095608@2x.png)
然后在pom中引入Spring Batch、MySQL和JDBC依赖，引入后pom内容如下所示：
```
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <parent>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-parent</artifactId>  
        <version>2.2.5.RELEASE</version>  
        <relativePath/> <!-- lookup parent from repository -->  
    </parent>  
    <groupId>cc.mrbird</groupId>  
    <artifactId>spring-batch-start</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <name>spring-batch-start</name>  
    <description>Demo project for Spring Boot</description>  
  
    <properties>  
        <java.version>1.8</java.version>  
    </properties>  
  
    <dependencies>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-batch</artifactId>  
        </dependency>  
  
        <dependency>  
            <groupId>mysql</groupId>  
            <artifactId>mysql-connector-java</artifactId>  
        </dependency>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-jdbc</artifactId>  
        </dependency>  
    </dependencies>  
  
    <build>  
        <plugins>  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
            </plugin>  
        </plugins>  
    </build>  
</project>
```
在编写代码之前，我们先来简单了解下Spring Batch的组成：
![QQ20200306-095955@2x](https://mrbird.cc/img/QQ20200306-095955@2x.png)
Spring Batch里最基本的单元就是任务Job，一个Job由若干个步骤Step组成。任务启动器Job Launcher负责运行Job，任务存储仓库Job Repository存储着Job的执行状态，参数和日志等信息。Job处理任务又可以分为三大类：数据读取Item Reader、数据中间处理Item Processor和数据输出Item Writer。

任务存储仓库可以是关系型数据库MySQL，非关系型数据库MongoDB或者直接存储在内存中，本篇使用的是MySQL作为任务存储仓库。

新建一个名称为springbatch的MySQL数据库，然后导入org.springframework.batch.core目录下的schema-mysql.sql文件：
![QQ20200306-102828@2x](https://mrbird.cc/img/QQ20200306-102828@2x.png)
导入后，库表如下图所示：
![QQ20200306-102603@2x](https://mrbird.cc/img/QQ20200306-102603@2x.png)
然后在项目的配置文件application.yml里添加MySQL相关配置：
```
spring:  
  datasource:  
    driver-class-name: com.mysql.cj.jdbc.Driver  
    url: jdbc:mysql://127.0.0.1:3306/springbatch  
    username: root  
    password: 123456
```
接着在Spring Boot的入口类上添加`@EnableBatchProcessing`注解，表示开启Spring Batch批处理功能：
```
@SpringBootApplication  
@EnableBatchProcessing  
public class SpringBatchStartApplication {  
    public static void main(String[] args) {  
        SpringApplication.run(SpringBatchStartApplication.class, args);  
    }  
}
```
至此，基本框架搭建好了，下面开始配置一个简单的任务。
## 编写第一个任务
在cc.mrbird.batch目录下新建job包，然后在该包下新建一个FirstJobDemo类，代码如下所示：
```
@Component  
public class FirstJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job firstJob() {  
        return jobBuilderFactory.get("firstJob")  
                .start(step())  
                .build();  
    }  
  
    private Step step() {  
        return stepBuilderFactory.get("step")  
                .tasklet((contribution, chunkContext) -> {  
                    System.out.println("执行步骤....");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
}
```
上面代码中，我们注入了`JobBuilderFactory`任务创建工厂和`StepBuilderFactory`步骤创建工厂，分别用于创建任务Job和步骤Step。`JobBuilderFactory`的`get`方法用于创建一个指定名称的任务，`start`方法指定任务的开始步骤，步骤通过`StepBuilderFactory`构建。

步骤Step由若干个小任务Tasklet组成，所以我们通过`tasklet`方法创建。`tasklet`方法接收一个`Tasklet`类型参数，`Tasklet`是一个函数是接口，源码如下：
```
public interface Tasklet {  
    @Nullable  
    RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception;  
}
```
所以我们可以使用lambda表达式创建一个匿名实现：
```
(contribution, chunkContext) -> {  
    System.out.println("执行步骤....");  
    return RepeatStatus.FINISHED;  
}
```
该匿名实现必须返回一个明确的执行状态，这里返回`RepeatStatus.FINISHED`表示该小任务执行成功，正常结束。

此外，需要注意的是，我们配置的任务Job必须注册到Spring IOC容器中，并且任务的名称和步骤的名称组成唯一。比如上面的例子，我们的任务名称为firstJob，步骤的名称为step，如果存在别的任务和步骤组合也叫这个名称的话，则会执行失败。

启动项目，控制台打印日志如下：
```
...  
2020-03-06 11:01:11.785 INFO 17324 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=firstJob]] launched with the following parameters: [{}]  
2020-03-06 11:01:11.846 INFO 17324 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
执行步骤....  
2020-03-06 11:01:11.886 INFO 17324 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step] executed in 40ms  
2020-03-06 11:01:11.909 INFO 17324 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=firstJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 101ms
```
可以看到，任务成功执行了，数据库的库表也将记录相关运行日志。

> 重新启动项目，控制台并不会再次打印出任务执行日志，因为Job名称和 Step名称组成唯一，执行完的不可重复的任务，不会再次执行。

## 多步骤任务

一个复杂的任务一般包含多个步骤，下面举个多步骤任务的例子。在job包下新建`MultiStepJobDemo`类：
```
@Component  
public class MultiStepJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job multiStepJob() {  
        return jobBuilderFactory.get("multiStepJob")  
                .start(step1())  
                .next(step2())  
                .next(step3())  
                .build();  
    }  
  
    private Step step1() {  
        return stepBuilderFactory.get("step1")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤一操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step2() {  
        return stepBuilderFactory.get("step2")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤二操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step3() {  
        return stepBuilderFactory.get("step3")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤三操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
}
```
上面代码中，我们通过`step1()`、`step2()`和`step3()`三个方法创建了三个步骤。Job里要使用这些步骤，只需要通过`JobBuilderFactory`的`start`方法指定第一个步骤，然后通过`next`方法不断地指定下一个步骤即可。

启动项目，控制台打印日志如下：
```
2020-03-06 13:52:52.188 INFO 18472 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=multiStepJob]] launched with the following parameters: [{}]  
2020-03-06 13:52:52.222 INFO 18472 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step1]  
执行步骤一操作。。。  
2020-03-06 13:52:52.251 INFO 18472 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step1] executed in 29ms  
2020-03-06 13:52:52.292 INFO 18472 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step2]  
执行步骤二操作。。。  
2020-03-06 13:52:52.323 INFO 18472 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step2] executed in 30ms  
2020-03-06 13:52:52.375 INFO 18472 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step3]  
执行步骤三操作。。。  
2020-03-06 13:52:52.405 INFO 18472 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step3] executed in 29ms  
2020-03-06 13:52:52.428 INFO 18472 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=multiStepJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 231ms
```
三个步骤依次执行成功。

多个步骤在执行过程中也可以通过上一个步骤的执行状态来决定是否执行下一个步骤，修改上面的代码：
```
@Component  
public class MultiStepJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job multiStepJob() {  
        return jobBuilderFactory.get("multiStepJob2")  
                .start(step1())  
                .on(ExitStatus.COMPLETED.getExitCode()).to(step2())  
                .from(step2())  
                .on(ExitStatus.COMPLETED.getExitCode()).to(step3())  
                .from(step3()).end()  
                .build();  
    }  
  
    private Step step1() {  
        return stepBuilderFactory.get("step1")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤一操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step2() {  
        return stepBuilderFactory.get("step2")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤二操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step3() {  
        return stepBuilderFactory.get("step3")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤三操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
}
```
`multiStepJob()`方法的含义是：multiStepJob2任务先执行step1，当step1状态为完成时，接着执行step2，当step2的状态为完成时，接着执行step3。`ExitStatus.COMPLETED`常量表示任务顺利执行完毕，正常退出，该类还包含以下几种退出状态：
```
public class ExitStatus implements Serializable, Comparable<ExitStatus> {  
  
    /**  
     * Convenient constant value representing unknown state - assumed not  
     * continuable.  
     */  
    public static final ExitStatus UNKNOWN = new ExitStatus("UNKNOWN");  
  
    /**  
     * Convenient constant value representing continuable state where processing  
     * is still taking place, so no further action is required. Used for  
     * asynchronous execution scenarios where the processing is happening in  
     * another thread or process and the caller is not required to wait for the  
     * result.  
     */  
    public static final ExitStatus EXECUTING = new ExitStatus("EXECUTING");  
  
    /**  
     * Convenient constant value representing finished processing.  
     */  
    public static final ExitStatus COMPLETED = new ExitStatus("COMPLETED");  
  
    /**  
     * Convenient constant value representing job that did no processing (e.g.  
     * because it was already complete).  
     */  
    public static final ExitStatus NOOP = new ExitStatus("NOOP");  
  
    /**  
     * Convenient constant value representing finished processing with an error.  
     */  
    public static final ExitStatus FAILED = new ExitStatus("FAILED");  
  
    /**  
     * Convenient constant value representing finished processing with  
     * interrupted status.  
     */  
    public static final ExitStatus STOPPED = new ExitStatus("STOPPED");  
  
    ...  
}
```
启动项目，控制台日志打印如下：
```
2020-03-06 14:21:49.384 INFO 18745 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=multiStepJob2]] launched with the following parameters: [{}]  
2020-03-06 14:21:49.427 INFO 18745 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step1]  
执行步骤一操作。。。  
2020-03-06 14:21:49.456 INFO 18745 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step1] executed in 29ms  
2020-03-06 14:21:49.501 INFO 18745 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step2]  
执行步骤二操作。。。  
2020-03-06 14:21:49.527 INFO 18745 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step2] executed in 26ms  
2020-03-06 14:21:49.576 INFO 18745 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step3]  
执行步骤三操作。。。  
2020-03-06 14:21:49.604 INFO 18745 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step3] executed in 28ms  
2020-03-06 14:21:49.629 INFO 18745 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=multiStepJob2]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 238ms
```
## Flow的用法

Flow的作用就是可以将多个步骤Step组合在一起然后再组装到任务Job中。举个Flow的例子，在job包下新建`FlowJobDemo`类：
```
@Component  
public class FlowJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job flowJob() {  
        return jobBuilderFactory.get("flowJob")  
                .start(flow())  
                .next(step3())  
                .end()  
                .build();  
    }  
  
    private Step step1() {  
        return stepBuilderFactory.get("step1")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤一操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step2() {  
        return stepBuilderFactory.get("step2")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤二操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step3() {  
        return stepBuilderFactory.get("step3")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤三操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    // 创建一个flow对象，包含若干个step  
    private Flow flow() {  
        return new FlowBuilder<Flow>("flow")  
                .start(step1())  
                .next(step2())  
                .build();  
    }  
}
```
上面代码中，我们通过`FlowBuilder`将step1和step2组合在一起，创建了一个名为flow的Flow，然后再将其赋给任务Job。使用Flow和Step构建Job的区别是，Job流程中包含Flow类型的时候需要在`build()`方法前调用`end()`方法。

启动程序，控制台日志打印如下：
```
2020-03-06 14:36:42.621 INFO 18865 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=flowJob]] launched with the following parameters: [{}]  
2020-03-06 14:36:42.667 INFO 18865 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step1]  
执行步骤一操作。。。  
2020-03-06 14:36:42.697 INFO 18865 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step1] executed in 30ms  
2020-03-06 14:36:42.744 INFO 18865 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step2]  
执行步骤二操作。。。  
2020-03-06 14:36:42.771 INFO 18865 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step2] executed in 27ms  
2020-03-06 14:36:42.824 INFO 18865 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step3]  
执行步骤三操作。。。  
2020-03-06 14:36:42.850 INFO 18865 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step3] executed in 25ms  
2020-03-06 14:36:42.874 INFO 18865 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=flowJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 245ms
```
## 并行执行

任务中的步骤除了可以串行执行（一个接着一个执行）外，还可以并行执行，并行执行在特定的业务需求下可以提供任务执行效率。

将任务并行化只需两个简单步骤：

1.  将步骤Step转换为Flow；
2.  任务Job中指定并行Flow。

举个例子，在job包下新建`SplitJobDemo`类：
```
@Component  
public class SplitJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job splitJob() {  
        return jobBuilderFactory.get("splitJob")  
                .start(flow1())  
                .split(new SimpleAsyncTaskExecutor()).add(flow2())  
                .end()  
                .build();  
  
    }  
  
    private Step step1() {  
        return stepBuilderFactory.get("step1")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤一操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step2() {  
        return stepBuilderFactory.get("step2")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤二操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step3() {  
        return stepBuilderFactory.get("step3")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤三操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Flow flow1() {  
        return new FlowBuilder<Flow>("flow1")  
                .start(step1())  
                .next(step2())  
                .build();  
    }  
  
    private Flow flow2() {  
        return new FlowBuilder<Flow>("flow2")  
                .start(step3())  
                .build();  
    }  
}
```
上面例子中，我们创建了两个Flow：flow1（包含step1和step2）和flow2（包含step3）。然后通过`JobBuilderFactory`的`split`方法，指定一个异步执行器，将flow1和flow2异步执行（也就是并行）。

启动项目，控制台日志打印如下：
```
2020-03-06 15:25:43.602 INFO 19449 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=splitJob]] launched with the following parameters: [{}]  
2020-03-06 15:25:43.643 INFO 19449 --- [cTaskExecutor-1] o.s.batch.core.job.SimpleStepHandler : Executing step: [step3]  
2020-03-06 15:25:43.650 INFO 19449 --- [cTaskExecutor-2] o.s.batch.core.job.SimpleStepHandler : Executing step: [step1]  
执行步骤三操作。。。  
执行步骤一操作。。。  
2020-03-06 15:25:43.673 INFO 19449 --- [cTaskExecutor-2] o.s.batch.core.step.AbstractStep : Step: [step1] executed in 23ms  
2020-03-06 15:25:43.674 INFO 19449 --- [cTaskExecutor-1] o.s.batch.core.step.AbstractStep : Step: [step3] executed in 31ms  
2020-03-06 15:25:43.714 INFO 19449 --- [cTaskExecutor-2] o.s.batch.core.job.SimpleStepHandler : Executing step: [step2]  
执行步骤二操作。。。  
2020-03-06 15:25:43.738 INFO 19449 --- [cTaskExecutor-2] o.s.batch.core.step.AbstractStep : Step: [step2] executed in 24ms  
2020-03-06 15:25:43.758 INFO 19449 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=splitJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 146ms
```
可以看到step3并没有在step2后才执行，说明步骤已经是并行化的（开启并行化后，并行的步骤执行顺序并不能100%确定，因为线程调度具有不确定性）。

## 任务决策器

决策器的作用就是可以指定程序在不同的情况下运行不同的任务流程，比如今天是周末，则让任务执行step1和step2，如果是工作日，则之心step1和step3。

使用决策器前，我们需要自定义一个决策器的实现。在cc.mrbird.batch包下新建decider包，然后创建`MyDecider`类，实现`JobExecutionDecider`接口：
```
@Component  
public class MyDecider implements JobExecutionDecider {  
    @Override  
    public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {  
        LocalDate now = LocalDate.now();  
        DayOfWeek dayOfWeek = now.getDayOfWeek();  
  
        if (dayOfWeek == DayOfWeek.SATURDAY || dayOfWeek == DayOfWeek.SUNDAY) {  
            return new FlowExecutionStatus("weekend");  
        } else {  
            return new FlowExecutionStatus("workingDay");  
        }  
    }  
}
```
`MyDecider`实现`JobExecutionDecider`接口的`decide`方法，该方法返回`FlowExecutionStatus`。上面的逻辑是：判断今天是否是周末，如果是，返回`FlowExecutionStatus("weekend")`状态，否则返回`FlowExecutionStatus("workingDay")`状态。

下面演示如何在任务Job里使用决策器。在job包下新建`DeciderJobDemo`：
```
@Component  
public class DeciderJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private MyDecider myDecider;  
  
    @Bean  
    public Job deciderJob() {  
        return jobBuilderFactory.get("deciderJob")  
                .start(step1())  
                .next(myDecider)  
                .from(myDecider).on("weekend").to(step2())  
                .from(myDecider).on("workingDay").to(step3())  
                .from(step3()).on("*").to(step4())  
                .end()  
                .build();  
    }  
  
    private Step step1() {  
        return stepBuilderFactory.get("step1")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤一操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step2() {  
        return stepBuilderFactory.get("step2")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤二操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
    private Step step3() {  
        return stepBuilderFactory.get("step3")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤三操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }  
  
  
    private Step step4() {  
        return stepBuilderFactory.get("step4")  
                .tasklet((stepContribution, chunkContext) -> {  
                    System.out.println("执行步骤四操作。。。");  
                    return RepeatStatus.FINISHED;  
                }).build();  
    }
```
上面代码中，我们注入了自定义决策器`MyDecider`，然后在`jobDecider()`方法里使用了该决策器：
```
@Bean  
public Job deciderJob() {  
    return jobBuilderFactory.get("deciderJob")  
            .start(step1())  
            .next(myDecider)  
            .from(myDecider).on("weekend").to(step2())  
            .from(myDecider).on("workingDay").to(step3())  
            .from(step3()).on("*").to(step4())  
            .end()  
            .build();  
}
```
这段代码的含义是：任务deciderJob首先执行step1，然后指定自定义决策器，如果决策器返回weekend，那么执行step2，如果决策器返回workingDay，那么执行step3。如果执行了step3，那么无论step3的结果是什么，都将执行step4。

启动项目，控制台输出如下所示：
```
2020-03-06 16:09:10.541 INFO 19873 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=deciderJob]] launched with the following parameters: [{}]  
2020-03-06 16:09:10.609 INFO 19873 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step1]  
执行步骤一操作。。。  
2020-03-06 16:09:10.641 INFO 19873 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step1] executed in 32ms  
2020-03-06 16:09:10.692 INFO 19873 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step3]  
执行步骤三操作。。。  
2020-03-06 16:09:10.723 INFO 19873 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step3] executed in 31ms  
2020-03-06 16:09:10.769 INFO 19873 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [step4]  
执行步骤四操作。。。  
2020-03-06 16:09:10.797 INFO 19873 --- [ main] o.s.batch.core.step.AbstractStep : Step: [step4] executed in 27ms  
2020-03-06 16:09:10.818 INFO 19873 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [FlowJob: [name=deciderJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 256ms
```
因为今天是2020年03月06日星期五，是工作日，所以任务执行了step1、step3和step4。

## 任务嵌套

任务Job除了可以由Step或者Flow构成外，我们还可以将多个任务Job转换为特殊的Step，然后再赋给另一个任务Job，这就是任务的嵌套。

举个例子，在job包下新建`NestedJobDemo`类：
```
@Component  
public class NestedJobDemo {  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
    @Autowired  
    private JobLauncher jobLauncher;  
    @Autowired  
    private JobRepository jobRepository;  
    @Autowired  
    private PlatformTransactionManager platformTransactionManager;  
  
    // 父任务  
    @Bean  
    public Job parentJob() {  
        return jobBuilderFactory.get("parentJob")  
                .start(childJobOneStep())  
                .next(childJobTwoStep())  
                .build();  
    }  
  
  
    // 将任务转换为特殊的步骤  
    private Step childJobOneStep() {  
        return new JobStepBuilder(new StepBuilder("childJobOneStep"))  
                .job(childJobOne())  
                .launcher(jobLauncher)  
                .repository(jobRepository)  
                .transactionManager(platformTransactionManager)  
                .build();  
    }  
  
    // 将任务转换为特殊的步骤  
    private Step childJobTwoStep() {  
        return new JobStepBuilder(new StepBuilder("childJobTwoStep"))  
                .job(childJobTwo())  
                .launcher(jobLauncher)  
                .repository(jobRepository)  
                .transactionManager(platformTransactionManager)  
                .build();  
    }  
  
    // 子任务一  
    private Job childJobOne() {  
        return jobBuilderFactory.get("childJobOne")  
                .start(  
                    stepBuilderFactory.get("childJobOneStep")  
                            .tasklet((stepContribution, chunkContext) -> {  
                                System.out.println("子任务一执行步骤。。。");  
                                return RepeatStatus.FINISHED;  
                            }).build()  
                ).build();  
    }  
  
    // 子任务二  
    private Job childJobTwo() {  
        return jobBuilderFactory.get("childJobTwo")  
                .start(  
                    stepBuilderFactory.get("childJobTwoStep")  
                            .tasklet((stepContribution, chunkContext) -> {  
                                System.out.println("子任务二执行步骤。。。");  
                                return RepeatStatus.FINISHED;  
                            }).build()  
                ).build();  
    }  
}
```
上面代码中，我们通过`childJobOne()`和`childJobTwo()`方法创建了两个任务Job，这里没什么好说的，前面都介绍过。关键在于`childJobOneStep()`方法和`childJobTwoStep()`方法。在`childJobOneStep()`方法中，我们通过`JobStepBuilder`构建了一个名称为`childJobOneStep`的Step，顾名思义，它是一个任务型Step的构造工厂，可以将任务转换为“特殊”的步骤。在构建过程中，我们还需要传入任务执行器JobLauncher、任务仓库JobRepository和事务管理器PlatformTransactionManager。

将任务转换为特殊的步骤后，将其赋给父任务parentJob即可，流程和前面介绍的一致。

配置好后，启动项目，控制台输出如下所示：
```
2020-03-06 16:58:39.771 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=parentJob]] launched with the following parameters: [{}]  
2020-03-06 16:58:39.812 INFO 21588 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [childJobOneStep]  
2020-03-06 16:58:39.866 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=childJobOne]] launched with the following parameters: [{}]  
2020-03-06 16:58:39.908 INFO 21588 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [childJobOneStep]  
子任务一执行步骤。。。  
2020-03-06 16:58:39.940 INFO 21588 --- [ main] o.s.batch.core.step.AbstractStep : Step: [childJobOneStep] executed in 32ms  
2020-03-06 16:58:39.960 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=childJobOne]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 86ms  
2020-03-06 16:58:39.983 INFO 21588 --- [ main] o.s.batch.core.step.AbstractStep : Step: [childJobOneStep] executed in 171ms  
2020-03-06 16:58:40.019 INFO 21588 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [childJobTwoStep]  
2020-03-06 16:58:40.067 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=childJobTwo]] launched with the following parameters: [{}]  
2020-03-06 16:58:40.102 INFO 21588 --- [ main] o.s.batch.core.job.SimpleStepHandler : Executing step: [childJobTwoStep]  
子任务二执行步骤。。。  
2020-03-06 16:58:40.130 INFO 21588 --- [ main] o.s.batch.core.step.AbstractStep : Step: [childJobTwoStep] executed in 28ms  
2020-03-06 16:58:40.152 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=childJobTwo]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 75ms  
2020-03-06 16:58:40.157 INFO 21588 --- [ main] o.s.batch.core.step.AbstractStep : Step: [childJobTwoStep] executed in 138ms  
2020-03-06 16:58:40.177 INFO 21588 --- [ main] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=parentJob]] completed with the following parameters: [{}] and the following status: [COMPLETED] in 398ms
```
> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[https://github.com/wuyouzhuguli/SpringAll/tree/master/67.spring-batch-start](https://github.com/wuyouzhuguli/SpringAll/tree/master/67.spring-batch-start)。