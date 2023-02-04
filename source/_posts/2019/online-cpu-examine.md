---
title: 线上服务器CPU100&#37排查
date: 2019-10-12 20:16:33
tags:
  - CPU
categories:
  - 服务器
---

## 背景

当天晚上{% label info@购物车 %}项目所在的服务器{% label danger@CPU突然飙升 %}，就在不久刚刚修复了一个线上的问题，难道是修复这个问题引起的吗？

由于我没有`线上服务器`的登录权限，项目经理将线程栈信息导出来发给了我，最后定位了问题。是进入{% label primary@购物车列表 %}这个操作引起的，找到问题并修复，最后CPU利用率也恢复了正常。

网上有很多关于定位CPU100%的文章，我关注了很多公众号，每个公众号几乎都推送过类似的文章。所以说，这是一项必会的本领，这里我自己总结下关于服务器CPU达到100%的原因以及如何解决这类的问题，就拿这次事件举例并分析。

## 情景再现

{% note primary %}
本地机器复现线上情况
{% endnote %}

本地启动代码，修改`nginx`配置，将测试环境机器指向到我的本地电脑。模拟线上数据，开始做测试。

进入购物车列表几次之后，本地CPU的利用率已经达到了100%，这时候机器已经很卡了。

![20230205004550](https:image.codingoer.top/blog/20230205004550.jpg)

## 定位过程

{% note info %}
### 执行`top`命令
{% endnote %}

{% codeblock lang:bash line_number:false mark:1 %}
top -c
{% endcodeblock %}

![20230205004616](https:image.codingoer.top/blog/20230205004616.jpg)

因为线上一台服务器会部署很多应用，先确定是不是我们的Java程序。

结果相吻合,就是`6560`的Java进程

![20230205004637](https:image.codingoer.top/blog/20230205004637.jpg)

<!-- more -->

{% note info %}
### CPU利用率最高的`线程`
{% endnote %}

找到CPU利用率最高的`线程`,看它做了什么事情,执行`top -Hp 6560`,按大写的`P`排序

{% codeblock lang:bash line_number:false mark:1 %}
top -Hp 6560
{% endcodeblock %}

![20230205004657](https:image.codingoer.top/blog/20230205004657.jpg)

{% note info %}
### 转换为16进制
{% endnote %}

{% codeblock lang:bash line_number:false %}
printf "0x%x\n" 6833
{% endcodeblock %}

得到的结果是`0x2653`

{% note info %}
### 查看线程堆栈
{% endnote %}

使用`jstack`,可以大概确定了位置

{% codeblock lang:bash line_number:false %}
jstack 6560 | grep 0x2653 -A 30
{% endcodeblock %}

![20230205004721](https:image.codingoer.top/blog/20230205004721.jpg)

{% note info %}
### 导出线程堆栈
{% endnote %}

{% codeblock lang:bash line_number:false %}
jstack -l 6560 > 6560.stack
{% endcodeblock %}

## 如何解决

首先来看一下问题代码，第五行代码那里出现了{% label danger@死循环 %}，从而导致了这次事件。

这是一个计算分摊金额的方法，将{% label info@cents %}参数的值，分摊到每个商品里面。

{% codeblock 省略了部分代码 lang:java mark:5-7 %}
    private void parsCents(long cents, List<ProductVo> products, int productStatus) {
        Map<ProductVo, AtomicInteger> unionIdToCount = new HashMap<>();
        while (cents > 0) {
            for (ProductVo productVo : products) {
                if (productVo.getProductStatus() == productStatus) {
                    continue;
                }
                int alloc = 1;
                AtomicInteger count = unionIdToCount.get(productVo);
                if(count == null) {
                    count = new AtomicInteger(alloc);
                    unionIdToCount.put(productVo, count);
                } else {
                    count.addAndGet(alloc);
                }
                cents -= alloc;
                if (cents == 0) {
                    return;
                }
            }
        }
    }
{% endcodeblock %}

最后代码修改，将while循环中的continue代码删掉，在外部先过滤掉失效的商品

{% codeblock 省略了部分代码 lang:java %}
    private void parsCents(long cents, List<ProductVo> products, int productStatus) {
        if (CollectionUtils.isEmpty(products)) {
            log.error("Ignore parse cents, products is empty");
            return;
        }
        List<ProductVo> productVoList = new ArrayList<>(products.size());
        for (ProductVo product : products) {
            if (product.getProductStatus() != productStatus) {
                productVoList.add(product);
            }
        }

        Map<ProductVo, AtomicInteger> unionIdToCount = new HashMap<>();
        OUT:
        while (cents > 0) {
            for (ProductVo productVo : productVoList) {
                int alloc = 1;
                AtomicInteger count = unionIdToCount.get(productVo);
                if(count == null) {
                    count = new AtomicInteger(alloc);
                    unionIdToCount.put(productVo, count);
                } else {
                    count.addAndGet(alloc);
                }
                cents -= alloc;
                if (cents == 0) {
                    break OUT;
                }
            }
        }
    }
{% endcodeblock %}

## 总结

{% note danger no-icon %}
此次事件就是写代码不规范导致的
{% endnote %}

在写`while`循环的时候尤其要注意，看看有没有死循环出现的可能。如果不小心就会引发严重的后果