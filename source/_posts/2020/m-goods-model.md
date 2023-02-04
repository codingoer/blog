---
title: 记一次泛型引发的线上事故
date: 2020-02-07 20:10:58
tags:
 - Java
categories:
 - Java
---

{% cq %}一次修改项目中的模型，导致的线上大面积事故{% endcq %}

首先说明下我们公司使用的是{% label info@Nexus %}搭建的{% label default @Maven私服 %}，因为各个项目中的模型都是相互使用的。如果在一个项目中修改了某一个模型，那么就要把这个模型{% label danger@deploy %}到远程仓库中去，其他人在别的项目中使用这个模型的话，就重新导入下依赖，这样就可以使用对方在模型中修改的属性或者方法了。

比如，我在{% label success@商品项目 %}中，添加了一个商品查询结果的模型，里面定义了一些属性和方法，我这边改好相应的代码之后，将这个模型发布到了远程仓库。然后小李在{% label success@订单项目 %}中想使用这个模型，只要重新导入下依赖就可以使用了。

## 事件回顾

当天晚上结束办公之后，由于是在家办公，六点多就下班了，我正在吃饭。突然客服运营群里说线上的搜索挂了，吓得我饭都吃不下了😟😟.

这几天正在开发新的需求，并没有上线什么功能，怎么好端端的会出现问题呢。当看到群里的截图时，我才发现这是一个{% label danger@JSON转换异常 %},看着报错的信息我怎么这么熟悉呢，这不就是我这几天做需求改动的模型吗？🥵没时间多考虑，赶紧电话和我们项目经理确认下，他说他上线了一个小需求，我告诉他线上除了问题，代码马上回滚了，过了几分钟一切就恢复正常了。

## 事件的原因

{% note danger no-icon %}
正因为是我改动的这个模型，导致了线上搜索全部挂了。
{% endnote %}

这几天我正在改商品库相关的业务需求，由于这次添加了导购的商品，所以要添加一个搜索方法。只不过这部分代码之前都是存在的，是搜索商品用的，当时我就想能不能复用这部分代码。

因为我们的{% label info@搜索条目 %}都在一个索引里面，不同的{% label info@搜索条目 %}是用类型区分的，他们都有一个共同的父类。如下图

![20230205010205](https:image.codingoer.top/blog/20230205010205.jpg)

这次只不过是新加一种商品搜索类型，所以我将商品的共同属性抽取到一个父类里面，如下图

![20230205010239](https:image.codingoer.top/blog/20230205010239.jpg)

所以我将搜索方法全部改成{% label warning@泛型代码 %}

> Talk is cheap, Show me code
{% tabs search-method, 1 %}
<!-- tab 原始搜索方法 -->
```java 搜索DTO伪代码
public class AdvancedSearchGoodsDto extends SearchItemDto<SearchGoods> {
}
```

```java 搜索结果VO伪代码
public class GoodsSearchResultVo implements Serializable {
    private List<SearchGoods> items;
}
```

```java 搜索Service伪代码
public class SearchItemService {
    GoodsSearchResultVo searchGoodsAdvanced(AdvancedSearchGoodsDto dto);
}
```

```java 搜索Dao伪代码
public class SearchItemDao {
    GoodsSearchResultVo searchGoodsAdvanced(AdvancedSearchGoodsDto dto);
}
```
<!-- endtab -->

<!-- tab 泛型搜索方法 -->
```java 搜索DTO伪代码
public class AdvancedSearchGoodsDto<T extends SearchBaseGoods> extends SearchItemDto<T> {
}
```

```java 搜索结果VO伪代码
public class GoodsSearchResultVo<T extends SearchBaseGoods> implements Serializable {
    private List<T> items;
}
```

```java 搜索Service伪代码
public class SearchItemService {
    <T extends SearchBaseGoods> GoodsSearchResultVo<T>(AdvancedSearchGoodsDto<T> dto);
}
```

```java 搜索Dao伪代码
public class SearchItemDao {
    <T extends SearchBaseGoods> GoodsSearchResultVo<T>(AdvancedSearchGoodsDto<T> dto);
}
```
<!-- endtab -->
{% endtabs %}
<!-- more -->

这样代码内部只要做下版本兼容就可以了，两个搜索就可以**使用同一套代码了**。

然后，我又开始修改客户端项目，也就是移动端的项目。因为移动端的项目要**依赖这个项目的搜索结果**，所以我就把商品库这个项目的模型{%label danger@deploy%}到了私服仓库中，然后在移动端项目更新项目依赖，这时候就将{% label warning@泛型代码 %}的搜索结果更新下来了，然后继续做修改。

因为搜索结果模型改成了{% label warning@泛型代码 %}，所以接口序列化方式也要做一些调整。由于{% label danger@泛型的限制性 %}，泛型参数最终**在虚拟中会被替换成Object对象**，所以原始的直接序列化成对象肯定是有问题的，必须要做下代码兼容。也就是先变成JSONObject，再根据每一个的搜索类型，序列化成对应的搜索对象。

以上就是我针对这次需求做出的代码改动。但是由于我的疏忽，没有做{% label info@项目版本升级 %},由于这个模型我是直接{%label danger@deploy%}到了maven仓库中，导致了其他同事更新下来了这个模型，在上线的时候将此模型打包到线上，最后导致序列化出错，线上搜索出现崩溃。

## 泛型代码的优缺点

我们都知道使用泛型有两点好处

- 增强代码的可读性
- 增强代码的复用性

我就是出于代码复用的角度去修改代码，使用的泛型虽然可以解决了这个问题。但是由于{% label danger@泛型的局限性 %}也会给系统带来其他的一些影响。比如这次问题，使用了泛型代码，之前客户端序列化的方式也要做出相应的调整，如果其他项目使用到此模型，那么每一个项目都要做出修改，这样给其他系统也带来了不可预知的影响。

所以原始代码修改为泛型代码的时候，一定要慎重，慎重，再慎重。一定要考虑对其他项目的影响。🔥🔥

## 项目依赖模型的问题

此次故障原因是，移动端项目依赖商品库项目的模型jar包做不兼容更新时，未升级版本号，导致上线打包发布时引用新的不兼容的jar包，导致数据无法转换异常。

目前，项目依赖版本管理方式容易导致此类问题，有时候因为疏忽，未对不兼容版本的代码升级版本，直接发布替换，导致代码被发布上线，就会有机率出现明明没修改相关代码，引起相关功能故障问题。

因为现在使用的版本均为{% label success@非正式版本 %}，而是{% label info@快照版本 %},因为目前业务复杂，项目依赖密切，如果使用正式版本，频繁的升级版本也不好维护项目。

此次事件发生后，项目版本发布均由我们项目经理一人管控，如果项目之间的模型进行了修改，一律使用{% label info@install %}命令安装到本地maven仓库，而不是使用{% label danger@deploy %}发布到远端maven仓库。

## 总结

经过此次事故之后，我进行了深刻的反思。在开发过程中，没有考虑到大局，只看到了眼前的项目。修改代码太过随意，没有考虑到版本的兼容性。

以后我会加强这方面的认识，修改代码之前一定要想好，考虑对之前的代码有没有影响。