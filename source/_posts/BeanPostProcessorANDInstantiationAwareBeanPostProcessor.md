---
title: 深入理解Spring BeanPostProcessor & InstantiationAwareBeanPostProcessor
top_img: /images/eye.png
date: 2024-09-13 11:06:57
categories:
  - Spring
tags:
  - Spring BeanPostProcessor & InstantiationAwareBeanPostProcessor
keywords: Spring, BeanPostProcessor,InstantiationAwareBeanPostProcessor
---

## 深入理解Spring BeanPostProcessor & InstantiationAwareBeanPostProcessor
在[深入学习Spring Bean生命周期](https://mrbird.cc/Spring-Bean-Lifecycle.html)一节中，我们学习了Bean后置处理器BeanPostProcessor，用于在Bean初始化前后插入我们自己的逻辑（Bean增强，Bean代理等）。今天偶然接触到BeanPostProcessor的子类InstantiationAwareBeanPostProcessor，用于Bean实例化前后处理。本节记录两者的区别以及简单原理分析。
## 两者比较

Initialization为初始化的意思，Instantiation为实例化的意思。在Spring Bean生命周期中，实例化指的是创建Bean的过程，初始化指的是Bean创建后，对其属性进行赋值（populate bean）、后置处理等操作的过程，所以Instantiation执行时机先于Initialization。

### 类关系

先来看看BeanPostProcessor的类结构：
![QQ20201208-190911@2x](https://mrbird.cc/img/QQ20201208-190911@2x.png)

![QQ20201209-090854@2x](https://mrbird.cc/img/QQ20201209-090854@2x.png)

### 方法解析

1.  BeanPostProcessor
    
    -   `postProcessBeforeInitialization(Object bean, String beanName)`：bean：Bean实例；beanName：Bean名称。方法将在Bean实例的afterPropertiesSet方法或者自定义的init方法被调用前调用，此时Bean属性已经被赋值。方法返回原始Bean实例或者包装后的Bean实例，如果返回null，则后续的后置处理方法不再被调用。
    -   `postProcessAfterInitialization(Object bean, String beanName)`：bean：Bean实例；beanName：Bean名称。方法将在Bean实例的afterPropertiesSet方法或者自定义的init方法被调用后调用，此时Bean属性已经被赋值。方法返回原始Bean实例或者包装后的Bean实例，如果返回null，则后续的后置处理方法不再被调用。
2.  InstantiationAwareBeanPostProcessor
    
    -   `postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`：beanClass：待实例化的Bean类型；beanName：待实例化的Bean名称。方法作用为：在Bean实例化前调用该方法，返回值可以为代理后的Bean，以此代替Bean默认的实例化过程。返回值不为null时，后续只会调用BeanPostProcessor的 postProcessAfterInitialization方法，而不会调用别的后续后置处理方法（如postProcessAfterInitialization、postProcessBeforeInstantiation等方法）；返回值也可以为null，这时候Bean将按默认方式初始化。
    -   `postProcessAfterInstantiation(Object bean, String beanName)`：bean：实例化后的Bean，此时属性还没有被赋值；beanName：Bean名称。方法作用为：当Bean通过构造器或者工厂方法被实例化后，当属性还未被赋值前，该方法会被调用，一般用于自定义属性赋值。方法返回值为布尔类型，返回true时，表示Bean属性需要被赋值；返回false表示跳过Bean属性赋值，并且InstantiationAwareBeanPostProcessor的postProcessProperties方法不会被调用。

### 执行时机对比

为了验证实例化和初始化的先后顺序，我们新建一个SpringBoot项目，版本2.4.0，依赖如下所示：
```
<dependencies>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter</artifactId>  
    </dependency>  
</dependencies>
```
Spring入口类名称为`DemoApplication`。新建`MyBeanPostProcessor`实现BeanPostProcessor接口：
```
@Component  
public class MyBeanPostProcessor implements BeanPostProcessor {  
  
    @Override  
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
        if ("demoApplication".equals(beanName)) {  
            System.out.println("post processor before " + beanName + " initialization");  
        }  
        return bean;  
    }  
  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
        if ("demoApplication".equals(beanName)) {  
            System.out.println("post processor after " + beanName + " initialization");  
        }  
        return bean;  
    }  
  
}
```

因为对所有的Bean生效，所以为了方便观察输出，这里仅当Bean名称为`demoApplication`时才打印输出。

接着新建`MyBeanInstantiationPostProcessor`实现InstantiationAwareBeanPostProcessor接口：
```
@Component  
public class MyBeanInstantiationPostProcessor implements InstantiationAwareBeanPostProcessor {  
  
    @Override  
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {  
        if ("demoApplication".equals(beanName)) {  
            System.out.println("post process before " + beanName + " instantiation");  
        }  
        return null;  
    }  
  
    @Override  
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {  
        if ("demoApplication".equals(beanName)) {  
            System.out.println("post process after " + beanName + " instantiation");  
        }  
        return true;  
    }  
  
    @Override  
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {  
        if ("demoApplication".equals(beanName)) {  
            System.out.println("post process " + beanName + " properties");  
        }  
        return pvs;  
    }  
}
```
启动程序，输出如下所示：
```
post process before demoApplication instantiation  
post process after demoApplication instantiation  
post process demoApplication properties  
post processor before demoApplication initialization  
post processor after demoApplication initialization
```
如果将MyBeanInstantiationPostProcessor的postProcessAfterInstantiation方法返回值改为false，程序输出如下：
```
post process before demoApplication instantiation  
post process after demoApplication instantiation  
post processor before demoApplication initialization  
post processor after demoApplication initialization
```
## 原理解析

postProcessAfterInitialization和InstantiationAwareBeanPostProcessor的方法都和Bean生命周期有关，要分析它们的实现原理自然要从Bean的创建过程入手。Bean创建的入口为`AbstractAutowireCapableBeanFactory`的createBean方法，查看其源码：
![2020-12-09 10-32-44](https://mrbird.cc/img/2020-12-09%2010-32-44.png)

resolveBeforeInstantiation方法源码如下所示：
![2020年12月09日10-57-07](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8809%E6%97%A510-57-07.png)
上面方法返回的bean如果为空的话，`AbstractAutowireCapableBeanFactory`的createBean方法将继续往下执行doCreateBean方法：

![2020年12月09日11-04-18](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8809%E6%97%A511-04-18.png)

查看doCreateBean方法源码：
![2020年12月09日14-11-50](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8809%E6%97%A514-11-50.png)
其他部分和本节讨论内容关系不大（Bean生命周期其他部分），重点关注populateBean和initializeBean方法。查看populateBean方法源码：
![2020年12月09日14-30-56](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8809%E6%97%A514-30-56.png)




接着查看initializeBean方法源码：![2020年12月09日14-36-48](https://mrbird.cc/img/2020%E5%B9%B412%E6%9C%8809%E6%97%A514-36-48.png)


至此我们通过查看Bean生命周期相关源码弄清楚了BeanPostProcessor和InstantiationAwareBeanPostProcessor相关方法的执行时机以及原理。

上面源码的追踪其实不仅涉及到了BeanPostProcessor和InstantiationAwareBeanPostProcessor相关方法的执行时机以及原理，更是整个Bean生命周期创建过程，结合[Spring-Bean生命周期](https://mrbird.cc/Spring-Bean%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.html)这篇文章的流程再走一遍源码，你会对Bean的生命周期有更深的理解。

## 总结

下面通过一张流程图总结本文：

![QQ20201209-151738@2x](https://mrbird.cc/img/QQ20201209-151738@2x.png)

> **原文来源：** https://github.com/wuyouzhuguli/SpringAll.
> 本节源码链接：[http://mrbird.cc/深入理解Spring-BeanPostProcessor-InstantiationAwareBeanPostProcessor.html](http://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Spring-BeanPostProcessor-InstantiationAwareBeanPostProcessor.html "深入理解Spring BeanPostProcessor & InstantiationAwareBeanPostProcessor")

