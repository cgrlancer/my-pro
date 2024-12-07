---
layout: spring Boot
top_img: /images/eye.png
title: Spring Batch任务调度
date: 2024-09-12 19:27:34
categories:
  - Spring Boot
tags:
  - Spring Batch教程
keywords: Spring Boot, Spring Batch
---

## Spring Batch任务调度
在前面的例子中，我们配置的任务都是在项目启动的时候自动运行，我们也可以通过`JobLauncher`或者`JobOperator`手动控制任务的运行时机，这节记录下它们的用法。

## 框架搭建
新建一个Spring Boot项目，版本为2.2.4.RELEASE，artifactId为spring-batch-launcher，项目结构如下图所示：
![QQ20200312-095759@2x](https://mrbird.cc/img/QQ20200312-095759@2x.png)
剩下的数据库层的准备，项目配置，依赖引入和[Spring Batch入门](https://mrbird.cc/Spring-Batch%E5%85%A5%E9%97%A8.html)文章中的框架搭建步骤一致，这里就不再赘述。

此外，本节我们需要演示在Controller里通过`JobLauncher`或者`JobOperator`调度任务，所以我们还需在pom里引入web依赖：
```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-web</artifactId>  
</dependency>
```
然后准备个任务，用于后续测试。在cc.mrbird.batch包下新建job包，然后在该包下新建`MyJob`：
```
@Component  
public class MyJob{  
  
    @Autowired  
    private JobBuilderFactory jobBuilderFactory;  
    @Autowired  
    private StepBuilderFactory stepBuilderFactory;  
  
    @Bean  
    public Job job(){  
        return jobBuilderFactory.get("job")  
                .start(step())  
                .build();  
    }  
  
    private Step step(){  
        return stepBuilderFactory.get("step")  
                .tasklet((stepContribution, chunkContext) -> {  
                    StepExecution stepExecution = chunkContext.getStepContext().getStepExecution();  
                    Map<String, JobParameter> parameters = stepExecution.getJobParameters().getParameters();  
                    System.out.println(parameters.get("message").getValue());  
                    return RepeatStatus.FINISHED;  
                })  
                .listener(this)  
                .build();  
    }  
}
```
在`step()`方法中，我们通过执行上下文获取了key为`message`的参数值。

## JobLauncher
在cc.mrbird.batch包下新建controller包，然后在该包下新建`JobController`：
```
@RestController  
@RequestMapping("job")  
public class JobController {  
  
    @Autowired  
    private Job job;  
    @Autowired  
    private JobLauncher jobLauncher;  
  
    @GetMapping("launcher/{message}")  
    public String launcher(@PathVariable String message) throws Exception {  
        JobParameters parameters = new JobParametersBuilder()  
                .addString("message", message)  
                .toJobParameters();  
        // 将参数传递给任务  
        jobLauncher.run(job, parameters);  
        return "success";  
    }  
}
```
上面代码中，我们注入了`JobLauncher`和上面配置的`Job`，然后通过`JobLauncher`的`run(Job job, JobParameters jobParameters)`方法运行指定的任务Job，并且传递了参数。

要关闭Spring Batch启动项目自动运行任务的机制，需要在项目配置文件application.yml中添加如下配置：
```
spring:  
  batch:  
    job:  
      enabled: false
```
启动项目，在浏览器地址栏访问：[http://localhost:8080/job/launcher/hello](http://localhost:8080/job/launcher/hello)：
![QQ20200312-102457@2x](https://mrbird.cc/img/QQ20200312-102457@2x.png)

项目控制台日志打印如下：
```
2020-03-12 10:24:31.547 INFO 41266 --- [nio-8080-exec-4] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=job]] launched with the following parameters: [{message=hello}]  
2020-03-12 10:24:31.583 INFO 41266 --- [nio-8080-exec-4] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
hello  
2020-03-12 10:24:31.610 INFO 41266 --- [nio-8080-exec-4] o.s.batch.core.step.AbstractStep : Step: [step] executed in 27ms  
2020-03-12 10:24:31.632 INFO 41266 --- [nio-8080-exec-4] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=job]] completed with the following parameters: [{message=hello}] and the following status: [COMPLETED] in 76ms
```

此外，需要注意的是：同样的参数，同样的任务再次运行的时候将抛出`JobInstanceAlreadyCompleteException`异常，比如在浏览器中再次访问[http://localhost:8080/job/launcher/hello](http://localhost:8080/job/launcher/hello)，项目控制台日志打印如下：
```
org.springframework.batch.core.repository.JobInstanceAlreadyCompleteException: A job instance already exists and is complete for parameters={message=hello}. If you want to run this job again, change the parameters.  
at org.springframework.batch.core.repository.support.SimpleJobRepository.createJobExecution(SimpleJobRepository.java:131) ~[spring-batch-core-4.2.1.RELEASE.jar:4.2.1.RELEASE]  
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_231]  
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_231]  
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_231]  
at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_231]  
at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:344) ~[spring-aop-5.2.4.RELEASE.jar:5.2.4.RELEASE]  
...
```
所以我们在任务调度的时候，应避免参数重复。

## JobOperator
在`JobController`里添加一个新的端点：
```
@RestController  
@RequestMapping("job")  
public class JobController {  
  
    @Autowired  
    private JobOperator jobOperator;  
  
    @GetMapping("operator/{message}")  
    public String operator(@PathVariable String message) throws Exception {  
        // 传递任务名称，参数使用 kv方式  
        jobOperator.start("job", "message=" + message);  
        return "success";  
    }  
}
```
上面代码中，我们注入了`JobOperator`，`JobOperator`的`start(String jobName, String parameters)`方法传入的是任务的名称（任务在Spring IOC容器中的名称）,并且参数使用key-value的方式传递。

要通过任务名称获取到相应的Bean，还需要添加一个额外的配置。在cc.mrbird.batch包下新建configure包，然后在该包下新建`JobConfigure`：
```
@Configuration  
public class JobConfigure {  
  
    /**  
     * 注册JobRegistryBeanPostProcessor bean  
     * 用于将任务名称和实际的任务关联起来  
     */  
    @Bean  
    public JobRegistryBeanPostProcessor processor(JobRegistry jobRegistry, ApplicationContext applicationContext) {  
        JobRegistryBeanPostProcessor postProcessor = new JobRegistryBeanPostProcessor();  
        postProcessor.setJobRegistry(jobRegistry);  
        postProcessor.setBeanFactory(applicationContext.getAutowireCapableBeanFactory());  
        return postProcessor;  
    }  
}
```
如果没有这段配置，在任务调度的时候将报org.springframework.batch.core.launch.NoSuchJobException: No job configuration with the name [job] was registered。

启动任务，浏览器访问：[http://localhost:8080/job/operator/mrbird](http://localhost:8080/job/operator/mrbird)：

![QQ20200312-105134@2x](https://mrbird.cc/img/QQ20200312-105134@2x.png)
项目控制台日志打印如下：
```
这2020-03-12 10:51:20.174 INFO 41405 --- [nio-8080-exec-2] o.s.b.c.l.support.SimpleJobOperator : Checking status of job with name=job  
2020-03-12 10:51:20.183 INFO 41405 --- [nio-8080-exec-2] o.s.b.c.l.support.SimpleJobOperator : Attempting to launch job with name=job and parameters=message=mrbird  
2020-03-12 10:51:20.239 INFO 41405 --- [nio-8080-exec-2] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=job]] launched with the following parameters: [{message=mrbird}]  
2020-03-12 10:51:20.293 INFO 41405 --- [nio-8080-exec-2] o.s.batch.core.job.SimpleStepHandler : Executing step: [step]  
mrbird  
2020-03-12 10:51:20.324 INFO 41405 --- [nio-8080-exec-2] o.s.batch.core.step.AbstractStep : Step: [step] executed in 31ms  
2020-03-12 10:51:20.344 INFO 41405 --- [nio-8080-exec-2] o.s.b.c.l.support.SimpleJobLauncher : Job: [SimpleJob: [name=job]] completed with the following parameters: [{message=mrbird}] and the following status: [COMPLETED] in 83ms
```
![QQ20200312-105318@2x](https://mrbird.cc/img/QQ20200312-105318@2x.png)
具体可以自己尝试玩一玩。



> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[[https://github.com/wuyouzhuguli/SpringAll/tree/master/73.spring-batch-launcher](https://github.com/wuyouzhuguli/SpringAll/tree/master/73.spring-batch-launcher)。

