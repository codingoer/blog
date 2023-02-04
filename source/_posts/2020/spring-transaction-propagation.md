---
title: 记录一次Spring事务分析过程
date: 2020-03-26 21:00:02
tags:
 - Spring
categories:
 - Spring
---

## 事件背景

今天下午写代码，在调试的过程中遇到了一个很熟悉的异常

> Transaction rolled back because it has been marked as rollback-only
这个异常大家都非常熟悉，是由于当前事务要提交的时候，发现这个事务已经被标记成回滚的事务了，所以提交事务的时候会抛出此异常。

借此机会就复习下关于事务的配置和传播级别等知识点。

## 事务的配置

Spring框架中提供了两种配置事务的方法，一种是{%label primary @ 声明式事务%}，一种是{% label info @ 注解式事务 %}。

### 声明式事务

一般项目中我们都会配置声明式事务，它是基于**Spring Aop**来实现的。

{% tabs Spring Transaction %}

<!-- tab Spring Boot配置 -->
总的来说，首先定义切面,然后注入`transactionManager`，配置切入点和通知。将切入点和通知绑定在一起。

{% codeblock 事务配置伪代码 lang:java line_number:false %}
@Configuration
@Aspect
public class TransactionAdviceConfig {
    @Autowired
    private PlatformTransactionManager transactionManager;

    @Bean
    public TransactionInterceptor txAdvice() {
        TransactionInterceptor transactionInterceptor = new TransactionInterceptor();
        transactionInterceptor.setTransactionManager(transactionManager);
        transactionInterceptor.setTransactionAttributeSource(source);
        return transactionInterceptor;
    }

    @Bean
    public Advisor txAdviceAdvisor() {
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor();
        advisor.setPointcut(pointcut);
        advisor.setAdvice(txAdvice());
        return advisor;
    }
}
{% endcodeblock %}
<!-- endtab -->

<!-- tab Spring Framework配置 -->
基于XML文件配置

{% codeblock 事务配置伪代码 lang:xml line_number:false %}
<bean id="transactionManager"
        class="org.springframework.orm.hibernate5.HibernateTransactionManager"
        p:session-factory-ref="dbSessionFactory" />

<tx:advice id="dbTxAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="persist*" propagation="REQUIRED" />
    </tx:attributes>
</tx:advice>

<aop:config proxy-target-class="true">
    <aop:pointcut id="dbTxPointcut" expression="execution(* com.xx.xx.service..*Service.*(..))" />
    <aop:advisor advice-ref="dbTxAdvice" pointcut-ref="dbTxPointcut" />
</aop:config>
{% endcodeblock %}
<!-- endtab -->

{% endtabs %}

### 注解式事务

{% note danger no-icon %}
如果使用配置文件要开启注解

```xml
<tx:annotation-driven transaction-manager="cartTxManager"/>
```
{% endnote %}

这种也很常见，在特殊的方法处理中添加`@Transactional`注解

{% codeblock 注解式事务方法 lang:java line_number:false %}
@Override
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public Item getById(Long id) {
    Optional<Item> optional = dao.findById(id);
    return optional.orElse(null);
}
{% endcodeblock %}

## 查看方法中的事务

一般项目中都会配置{%label primary @ 声明式事务%}，这样可以全局处理，不用为每一个方法单独配置事务。但是再处理一些特殊的业务时，会需要加上{% label info @ 注解式事务 %}覆盖原来的事务，上一层的事务会被挂起，等当前的事务提交之后再执行上一层事务。

可以通过**Debug**的方式来查看当前方法一共有几层事务，在业务方法中打个断点。

看看有几次`invokeWithinTransaction`，如下图所示，那说明当前方法中有两层事务。

- 第一次
![20230205010816](https:image.codingoer.top/blog/20230205010816.jpg)

- 第二次
![20230205010837](https:image.codingoer.top/blog/20230205010837.jpg)

## 事务的传播级别

关于事务的传播级别可以看源码`Propagation`，每一种情况都要自己好好分析，跑一个单列测试。下面我就列举几个比较常用的

- REQUIRED

支持当前的事务，如果当前没有事务会创建一个新的事务。

- SUPPORTS

支持当前的事务，如果没有事务就不开启事务。

- REQUIRES_NEW

创建一个新的事务，如果当前有事务会挂起当前事务。

- NOT_SUPPORTED

不使用事务，如果当前有事务会挂起当前事务。

## 异常的原因

回到最开始的问题，这个`Transaction rolled back because it has been marked as rollback-only`异常是如何产生的。

{% codeblock 问题代码 lang:java line_number:false %}
public void saveOrUpdate(List<BrandDtk> brandList) {
    for (DtkBrandVo dtkBrandVo : brandList) {
        BrandDtk brandDtk = service.get(brandId);
        if (brandDtk == null) {
            brandDtk = new BrandDtk();
            brandDtk.setId(brandId);
            try {
                service.save(toBrand(brandDtk, dtkBrandVo));
            } catch (ConstraintViolationException e) {

            }
        } else {
            service.update(toBrand(brandDtk, dtkBrandVo));
        }
    }
}
{% endcodeblock %}

注解配置中将**Save**开头的方法的传递级别设置成{%label danger@REQUIRED%},也就是说`saveOrUpdate`方法会开始一个事务，方法内如果有事务延用当前的事务。

同理，方法内部的`save`方法也会开始一个事物，但是外部有事务，会使用外部的事务。也就是说`saveOrUpdate`方法和`save`方法使用的是同一个事务。

由于唯一索引冲突，`save`方法抛出异常，但是在`saveOrUpdate`方法中捕获的异常，看起来没什么问题。但是`save`方法抛出异常的同时将事务标记成了{% label danger@回滚状态%},由于两个方法是同一个事务，所以`saveOrUpdate`方法在提交事务的时候会跑出此异常。

真正的异常是唯一索引冲突异常，但是在内部已经被吃掉了。

## 总结

解决这个问题，只要在方法上将事务改为{%label danger@NOT_SUPPORTED%}即可。

在写业务方法的时候，一定要考虑清楚当前场景是否需要使用事务，以及如何使用事务的传播级别。