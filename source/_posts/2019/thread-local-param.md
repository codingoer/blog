---
title: 多线程中ThreadLocal的传参
date: 2019-11-21 18:12:09
categories:
 - Java 
tags: 
 - Java
---

APP客户端发起Http请求，服务端这边执行业务方法。将一些信息比如参数等放入`ThreadLocal`中保存，后续的方法可以随时获取这个参数，这样对代码的侵入性很小。

如果在业务方法中，使用{% label info@线程池 %}来处理其他业务，加速服务端相应时间。那么在线程池中如何获取到`Http线程`中存放的参数呢。

{% note primary  %}
可以通过{% label success @**Callable + Future** %}来实现
{% endnote %}

## 代码实现

App发起请求，进入业务方法，此时已经将参数放入`ThreadLocal`中

{% codeblock service方法 lang:java line_number:false %}
@Override
public List<Module> getModuleByIds(Long[] moduleIds) {
    List<ModulePage> modulePages = dao.get(moduleIds);
    try {
        List<Callable<Module>> callables = new ArrayList<>(modulePages.size());
        modulePages.forEach(modulePage -> {
            callables.add(new ModuleCallable(modulePage));
        });
        List<Module> rModuleList = PoolUtils.execute(callables);
        TimeCountUtils.escape("execute");
    } finally {
        ThreadLocalUtils.clear();
    }
    return modules;
}
{% endcodeblock %}

创建一个`ModuleCallable`对象，实现`Callable`接口，构造方法中从Http线程中获取参数并赋值。

{% codeblock ModuleCallable对象代码实现 lang:java line_number:false %}
public class ModuleCallable implements Callable<Module> {

    private Map<String, String> requestHeaders;

    public ModuleCallable(ModulePage modulePage) {
        requestHeaders = HttpRequestUtils.requestHeaders();
    }

    @Override
    public Module call() {
        HttpRequestUtils.requestHeaders(requestHeaders);
        try {
            return ModuleParseUtils.parse(modulePage);
        } finally {
            HttpRequestUtils.clear();
        }
    }
}
{% endcodeblock %}

在执行`call`回调方法中，将本地参数赋值到线程池中的线程的`ThreadLocal`中，这样就可以在后续方法中，再次获取到这个参数。

## 注意点

无论是在http请求中还是在线程池中，处理完业务最终都要释放`ThreadLocal`中的参数，以免被其他线程复用，导致业务错误。