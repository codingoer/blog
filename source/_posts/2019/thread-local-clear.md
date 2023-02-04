---
title: 记一次ThreadLocal导致的问题
date: 2019-08-07 18:02:58
categories:
 - Java 
tags: 
 - Java
---

## 前言

当天测试反馈一个{% label danger@Bug %}🐞，是我们APP首页有一个模型，正常情况是要根据每个用户浏览的商品来推荐一些相关的商品。首页的这个模型默认展示前两个商品，点进去会进入到推荐商品列表页,看下图

{% grouppicture 2-2 %}
    {% img https://image-m.1haitao.com/m/50881151504359879833.png %}
    {% img https://image.codingoer.top/blog/20230205011447.jpg %}
{% endgrouppicture %}

测试发现，每次刷新首页的时候，这两个商品有时候会变化，与点击进去的列表页的头两个商品对不上。

## 定位问题

{% note warning %}
出现问题不要慌,要找准问题的关键点.
{% endnote %}

首先,这个模型展示的商品是根据{% label info@用户 %}来决定的,不同的用户所看到的商品是不一样的.进入列表页之后看到的商品列表也是根据{% label info@用户 %}来决定的.

那现在出现的问题是

1. 首页这个模型上展示的商品,刷新首页会不停变化
2. 进入到列表页和列表页的商品对不上.

经过上面的分析之后,可以大概知道是根据用户ID查询的数据出现了问题.多半就是由于这个用户ID引起的,要不为什么数据会变化呢??有可能首页的模型数据是别的用户的,而列表页的数据是自己的.

所以就按照这个方向来定位问题,那么就开始检查代码吧,先看下存储用户信息的这部分代码

因为我们的服务部署在{% label success@Tomcat %}服务器上,{% label info@Tomcat %}又是基于`线程`的.所以我们后端服务将用户信息放在了{% label success@ThreadLocal %}中.

```java  用户工具类伪代码
public class UserServletUtils {
    private static ThreadLocal<User> userCache = new ThreadLocal<>();
    public static User user() {
        User user = userCache.get();
        if (user != null) {
            return user;
        }
        user = ucenterService.user();
        if (user != null) {
            userCache.set(user);
            return user;
        }
    }
}
```

这样用户的信息就**贯穿整个请求**,后面用到用户信息就直接工具类获取,代码写起来**更简单**.  
但是这样用户信息就和当前线程绑定了,随着线程而消亡,这点非常重要,所以我们每次请求之后都会清空本地线程栈的信息

```java servlet伪代码
public class DispatcherServlet extends ContextDispatcherServlet {
    @Override
    protected void doService() {
        try {
            super.doService(request, response);
        } catch (Exception e) {
            throw e;
        } finally {
            UserServletUtils.clear();
            super.clear();
        }
    }
}
```

<!-- more -->

---

再看下首页加载模型的代码.因为首页涉及到多个模型,很自然代码使用了{% label primary@多线程 %}的方式.

先根据页面ID查询出来这个页面都有哪些模型,代码如下

```java 首页加载模型伪代码
public class WebPageService() {
    @Autowired
    private IModuleService moduleService;
    
    public Web getById() {
        Web web = new Web();
        List<Module> moduleList = moduleService.getModuleByIds(ids);
        web.setModules(moduleList);
    }
}
```

得到这些模型id之后,开始批量获取这些模型.首先定义了`callable`对象,将一个个`callable`对象放到集合中.`PoolUtils`内部封装了`ExecutorService`,底层使用的是`Callable` + `Future`.

```java 模型处理伪代码
public class ModuleService() {
    public List<Module> getModuleByIds(Long[] moduleIds) {
        List<ModulePage> modulePages = dao.get(moduleIds);
        List<Callable<Module>> callables = new ArrayList<>(modulePages.size());
        modulePages.forEach(modulePage -> {
            if (ModuleParseUtils.effectiveModule(modulePage)) {
                callables.add(new ModuleCallable(modulePage));
            }
        });
        List<Module> rModuleList = PoolUtils.execute(callables);
    }
}
```

```java Callable伪代码
public class ModuleCallable {
    private ModulePage modulePage;
    public ModuleCallable(ModulePage modulePage) {
        this.modulePage = modulePage;
    }
    @Override
    public Module call() {
        return ModuleParseUtils.parseByPhp(modulePage, null, null);
    }
}
```

最后解析这些模型,使用的是{% label info@工厂模式 %},做成统一抽象.

---

代码从头到位看下来.其实并不复杂.只要看准两个关键点.

- 用户的存储方式是ThreadLocal
- 模型的加载方式是多线程

如果是经验丰富的人,可能到现在已经知道问题的所在了.下面我们继续通过代码调试看

## 代码调试

{% note info %}
打几个断点一看便知
{% endnote %}

进入到获取模型处打一个断点,我们可以看到当前线程是`http-bio-8164-exec-1`,这就是我们通过Tomcat请求进来的线程.

![20230205011254](https:image.codingoer.top/blog/20230205011254.jpg)

回顾刚才说的,我们将用户ID放在了这个线程的本地栈中,然后等请求结束后又清除了这次请求的用户信息.在本次请求内,可以随时随地通过工具类获取到用户信息.真是完美.💥

进入到模型里面打一个断点,我们可以看到当前线程是`pool-30-thread-5`,这是线程池里面的线程.

![20230205011356](https:image.codingoer.top/blog/20230205011356.jpg)

回顾刚才说的,因为模型这部分我们使用的是多线程处理,所以这部分处理都在线程池中的线程处理.如果获取用户信息的话,还是放在本次线程中.只不过使用完之后并没有清除线程栈中的信息.

下次请求的时候还会复用当前线程中的用户信息,这就是导致本次事件的原因.

## 解决问题

知道了问题所在,那么这个问题就很解决了,只需要释放线程池中线程存储的用户信息就可以了.

```java Callable伪代码
public class ModuleCallable {
    private ModulePage modulePage;
    public ModuleCallable(ModulePage modulePage) {
        this.modulePage = modulePage;
    }
    @Override
    public Module call() {
        try {
            return ModuleParseUtils.parseByPhp(modulePage, null, null);
        } finally {
            UserServletUtils.clear();
        }
    }
}
```

## 总结

本次问题是使用{% label info @多线程 %} + {% label info @ThreadLocal%}不当造成的问题.

在使用{% label info @ThreadLocal%}的时候尤其要注意,因为它是与线程绑定的,比如线程池中的线程都是采取复用的方法,那么这些线程一般永远不会结束.这也就意味着{% label info @ThreadLocal%}中的对象永远也不会消失,这些对象会占据大量空间.这样会造成无法预计的后果.可能会导致数据混乱,可能会导致系统崩溃.

所以,我们在平时开发中,如果使用ThreadLocal就一定要注意,业务处理完一定要清除其中存储的对象.