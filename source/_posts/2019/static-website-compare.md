---
title: 静态网站生成器对比
date: 2019-07-14 16:18:06
tags:
  - SSG
categories:
  - Website
---

## 前言
最开始因为[搭建博客](/2019/06/28/about-my-blog/)，上网查阅了很多关于{% label info@静态网站生成器 %}的相关文章，后续自己又尝试了几款不同的静态网站生成器，并对其有了一定的了解。

- 我的博客是用[Hexo](https://hexo.io/zh-cn/)搭建的
- 我的笔记本是用[Vuepress](https://vuepress.vuejs.org/)搭建的
- 公司的笔记本是用[Docsify](https://docsify.js.org/#/)搭建的

先截图看下

{% tabs SSGs example, 1 %}
<!-- tab Hexo -->
{% img https://image.codingoer.top/blog/20230205012356.jpg %}

<!-- endtab -->

<!-- tab Vuepress -->
{% img https://image.codingoer.top/blog/20230205012446.jpg %}

<!-- endtab -->

<!-- tab Docsify -->
{% img https://image.codingoer.top/blog/20230205012525.jpg %}
<!-- endtab -->
{% endtabs %}

{% note primary no-icon%}
下面主要介绍下静态网站生成器，并主要说明我试过的这几个，做一下对比
{% endnote %}

## 什么是静态网站生成器

> 静态网站 - static site
**静态网站** 是包含html页面的集合

> 静态网站生成器 - static site generator(SSG)
**静态网站生成器** 使用原始数据（比如Markdown文件）和模板，产生Html页面的工具。

我个人的理解，主要是将{% label danger@markdown %}渲染成`html`页面，并根据`模板`生成一些静态页面。

找了一张网络上的图片，很形象

![20230205012245](https:image.codingoer.top/blog/20230205012245.jpg)

从图片上我们不难理解，将一些资源文件，这些文件包括`布局`,`javascript`,`css`,`image`,`md文件`等等，最终生成`html页面`.

<!-- more -->

## 静态网站生成器对比

关于{% label info@静态网站生成器 %}的种类真的是太多太多了。耳熟能详的就有，{% label info@Jekyll %}，{% label info@Hugo %}，{% label info@Hexo %}，{% label info@Gatsby %}........

> 按照所基于语言划分
- 基于React(JavaScript)的系列: `Gatsby` `Next.js`
- 基于VUE的(JavaScript)系列: `Vuepress` `Nuxt.js`
- 基于Ruby的系列: `JekkyII`
- 基于GoLang的系列: `Hugo`

> 按照目的划分
- 博客或者小型个人网站：`Hexo`，`Gatsby`
- 文档：`GitBook`，`Vuepress`
- 商业网站：`Gatsby`，`Nuxt`

{% note info %}
下面对比我使用过的三个，{%label success@Hexo %} VS {%label success@Vuepress %} VS {%label success@Docsify %}
{% endnote %}

### Hexo
我的博客就是使用{% label info@Hexo %}搭建的，总体来说，我觉得{% label info@Hexo %}国内使用的人比较多，中文资料多，百度一搜问题，可以搜出来相关的问题。然后就是上手容易，照着文档上操作，遇到问题都可以解决。

主题比较丰富，我Google了Top 10主题，每一个都看了。下面几个主题都是我比较喜欢的，也是排名比较靠前的。

- [next](https://github.com/theme-next/hexo-theme-next)
- [icarus](https://github.com/ppoffice/hexo-theme-icarus)
- [snippet](https://github.com/shenliyang/hexo-theme-snippet)

官网提供的插件也是比较多，但是我没有用，这里不多说了。

{% note danger no-icon %}
缺点就是出现问题不好定位，错误信息不明显，也可能是我不会调试吧，毕竟前端不是很好
{% endnote %}

比如我在标题中输入了特殊符号`%`

```frontmatter
---
title: 我是特殊符号%
---
```

使用`hexo server`命令时会出现以下错误信息

```javascript
ERROR Render HTML failed: index.html
TypeError: Cannot read property 'replace' of null
    at Hexo.externalLinkFilter (/Users/lionel/GitProjects/blog/node_modules/hexo/lib/plugins/filter/after_render/external_link.js:22:15)
    at Hexo.tryCatcher (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/util.js:16:23)
    at Hexo.<anonymous> (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/method.js:15:34)
    at Promise.each.filter (/Users/lionel/GitProjects/blog/node_modules/hexo/lib/extend/filter.js:62:52)
    at tryCatcher (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/util.js:16:23)
    at Object.gotValue (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/reduce.js:166:18)
    at Object.gotAccum (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/reduce.js:155:25)
    at Object.tryCatcher (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/promise.js:547:31)
    at Promise._settlePromise (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/promise.js:604:18)
    at Promise._settlePromiseCtx (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/promise.js:641:10)
    at _drainQueueStep (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/async.js:97:12)
    at _drainQueue (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/async.js:86:9)
    at Async._drainQueues (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/async.js:102:5)
    at Immediate.Async.drainQueues [as _onImmediate] (/Users/lionel/GitProjects/blog/node_modules/bluebird/js/release/async.js:15:14)
    at runCallback (timers.js:705:18)
    at tryOnImmediate (timers.js:676:5)
    at processImmediate (timers.js:658:5)
```

头疼的很，完全看不出来是标题引起的问题，只能一点点排查，最后定位问题。

### Vuepress
{% label info@Vuepress %}更适合做文档，VueJs生态的产品都是使用它做的官网文档。它提供了一个自带的主题，适合写技术文档，也可以自己开发主题。

因为是基于`VueJs`的，调试起来比较方便，出现错误信息马上能定位位置，适合熟悉`VueJs`的人上手。

{% label info@Vuepress %}内部集成了webpack，还有node的相关知识，感觉逼格挺高，弄起来确实要花点时间。

我在弄我的笔记的时候确实花了不少时间，看了很多遍的文档，也操作了很多遍。有时间想自己写写主题或者插件，都是基于vue的

### Docsify
{% note success %}
最大的优点就是不用生成html页面
{% endnote %}

{% label info@Vuepress %}更小巧吧，首页就是一个html页面，所有的配置都在这里面配置，直接写md页面就可以了。然后通过nginx指向这个首页就可以访问了，真的是很方便。因为我公司的笔记本无法公开，所以前面只能截了个图，大家可以看看官网，上手也是很快的。

### 一些共同特点

每一种静态网站生成器基本上都是一样的，我们只需要关心如何写作，其他的东西只要按照官方文档一步步来就可以了。

除了基本的Markdown语法外，不同的生成器还会支持一些特有的语法，比如`lable`,`note`等等。大同小异，就是样式不一样罢了。所以每一种静态网站生成器都差不多，不要在这上面花太多时间。我就是在这上面浪费了比较多的时间，最后发现也没什么卵用。

## 如何选择适合你的

{% note danger %}
**主题和插件** 占据主导地位，然后就看个人喜好了
{% endnote %}

首先要明确一点，你搭建这个静态网站做什么用？是写博客，还是写技术文档，还是商业使用。明确了这一点后，才能选择适合你的。

如果写文档我推荐使用{% label info@Vuepress %}，毕竟`VueJs`还是比较流行的，大多数人都了解，或者使用{% label info@Docsify %}，也适合写文档。如果是商业使用，我推荐使用Gatsby，搭建博客的话，还是选择国内使用比较多的，你遇到的问题基本上大家都遇到过，问题好解决，这样少走弯路，尽快达到目的。

其次还要看你对前端哪块比较熟悉，如果你熟悉`React`那么肯定要选择基于`React`的系列(Gatsby，Next.js)，如果你熟悉`VueJs`，那么肯定选择基于`VueJs`的系列(Vuepress, Nuxt.js)，这样上手更加容易，事半功倍。

我们最终的目的是搭建完成，既要考虑美观，也要考虑学习成本，尽快完成才是我们的目标。尽量不要给自己挖坑，最后半途而废！！

## 总结

搭建静态网站并不难，我们看到各种各样的{% label info@静态网站生成器 %}也不必感到无从下手，万变不离其中，所有的原理都是一样的。唯一不同的就是官方文档不同，只要你通读了官网文档，并且边看边做，你就学会了。

最后要的就是你要用这个东西做什么事，不能说搭建完成就放在那里了，然后就荒废了。网站搭建完成只是一个开始，路漫漫其修远兮。最重要的是做事和坚持。

最后来一句东北话，干就完了！！