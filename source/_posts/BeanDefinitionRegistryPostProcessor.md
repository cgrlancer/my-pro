---
title: 深入理解BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor
top_img: /images/eye.png
date: 2024-09-13 11:06:57
categories:
  - Spring
tags:
  - Spring 深入理解BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor
keywords: Spring, 深入理解BeanFactoryPostProcessor,BeanDefinitionRegistryPostProcessor
---
## 深入理解BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor
本节主要记录BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor的方法执行时机以及简单原理分析。

## BeanFactoryPostProcessor
查看BeanFactoryPostProcessor源码：
![QQ20201224-104608@2x](https://mrbird.cc/img/QQ20201224-104608@2x.png)
根据注释我们了解到postProcessBeanFactory方法的执行时机为：BeanFactory标准初始化之后，所有的Bean定义已经被加载，但Bean的实例还没被创建（不包括BeanFactoryPostProcessor类型）。该方法通常用于修改bean的定义，Bean的属性值等，甚至可以在此快速初始化Bean。

下面测试一波。

新建SpringBoot项目，Boot版本2.4.0，依赖如下：
```
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter</artifactId>  
</dependency>
```
然后新建MyBeanFactoryPostProcessor，实现BeanFactoryPostProcessor接口：
```
@Component  
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {  
  
    private static final Logger logger = LoggerFactory.getLogger(MyBeanFactoryPostProcessor.class);  
  
    public MyBeanFactoryPostProcessor() {  
        logger.info("实例化MyBeanFactoryPostProcessor Bean");  
    }  
  
    @Override  
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {  
        int beanDefinitionCount = beanFactory.getBeanDefinitionCount();  
        logger.info("Bean定义个数: " + beanDefinitionCount);  
    }  
  
    @Component  
    static class TestBean {  
        public TestBean() {  
            logger.info("实例化TestBean");  
        }  
    }  
}
```
在postProcessBeanFactory方法内，我们打印了当前已加载Bean定义的个数，并且在MyBeanFactoryPostProcessor类中，注册了TestBean。MyBeanFactoryPostProcessor和TestBean的构造函数输出的日志用于观察Bean实例化时机。

启动程序，输出如下：
![QQ20201224-105343@2x](https://mrbird.cc/img/QQ20201224-105343@2x.png)

上面的日志证实了方法的执行时机的确是在BeanFactory标准初始化之后，所有的Bean定义已经被加载，但Bean的实例还没被创建（此时TestBean还未被实例化，日志还没有输出”实例化TestBean”，但这不包括BeanFactoryPostProcessor类型Bean，该方法执行之前，日志就已经输出了”实例化MyBeanFactoryPostProcessor Bean”）。

我们在postProcessBeanFactory方法上打个断点：
![QQ20201224-105621@2x](https://mrbird.cc/img/QQ20201224-105621@2x.png)

以debug方式启动程序：
![QQ20201224-110850@2x](https://mrbird.cc/img/QQ20201224-110850@2x.png)


通过追踪方法调用栈，我们可以总结出BeanFactoryPostProcessor的postProcessBeanFactory方法执行时机和原理：

1.  `SpringApplication.run(MyApplication.class, args)`启动Boot程序：
    
    ![QQ20201224-143234@2x](https://mrbird.cc/img/QQ20201224-143234@2x.png)
    
2.  `run`方法内部调用`refreshContext`方法刷新上下文：
    
    ![QQ20201224-143311@2x](https://mrbird.cc/img/QQ20201224-143311@2x.png)
    
3.  `refresh`方法内部调用`invokeBeanFactoryPostProcessors`方法：
    
    ![QQ20201224-143410@2x](https://mrbird.cc/img/QQ20201224-143410@2x.png)
    
4.  PostProcessorRegistrationDelegate的`invokeBeanFactoryPostProcessors`方法内部：
    
    ![2020年12月24日14-41-11](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8824%E6%97%A514-41-11.png)
    
    ![QQ20201224-144317@2x](https://mrbird.cc/img/QQ20201224-144317@2x.png)

## BeanDefinitionRegistryPostProcessor

BeanDefinitionRegistryPostProcessor继承自BeanFactoryPostProcessor，新增了一个postProcessBeanDefinitionRegistry方法：

![QQ20201224-153034@2x](https://mrbird.cc/img/QQ20201224-153034@2x.png)

通过注释我们了解到postProcessBeanDefinitionRegistry方法的执行时机为：所有的Bean定义即将被加载，但Bean的实例还没被创建时。也就是说，BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法执行时机先于BeanFactoryPostProcessor的postProcessBeanFactory方法。这个方法通常用于给IOC容器添加额外的组件。

举个例子测试一波。

新建BeanDefinitionRegistryPostProcessor的实现类MyBeanDefinitionRegistryPostProcessor：
```
@Component  
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {  
  
    private final Logger logger = LoggerFactory.getLogger(MyBeanDefinitionRegistryPostProcessor.class);  
  
    @Override  
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {  
        int beanDefinitionCount = registry.getBeanDefinitionCount();  
        logger.info("Bean定义个数: " + beanDefinitionCount);  
        // 添加一个新的Bean定义  
        RootBeanDefinition definition = new RootBeanDefinition(Object.class);  
        registry.registerBeanDefinition("hello", definition);  
    }  
  
    @Override  
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {  
  
    }  
}
```
启动程序，输出如下：

![QQ20201224-153548@2x](https://mrbird.cc/img/QQ20201224-153548@2x.png)

可以看到，BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法执行时机的确先于BeanFactoryPostProcessor的postProcessBeanFactory方法。

通过查看PostProcessorRegistrationDelegate的`invokeBeanFactoryPostProcessors`方法源码也可以证实这一点：

![2020年12月24日15-44-15.png](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8824%E6%97%A515-44-15.png)



> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[http://mrbird.cc/深入理解BeanFactoryPostProcessor-BeanDefinitionRegistryPostProcessor.html](http://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3BeanFactoryPostProcessor-BeanDefinitionRegistryPostProcessor.html "深入理解BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor")
