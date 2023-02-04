---
title: 使用Markdown写作的一些事
date: 2019-07-18 20:30:02
tags:
 - Markdown
categories:
 - Website
---

因为我是使用{%label info@hexo %} + {% label success@next %}构建的网站，所以在这里我总结下{%label info@hexo %}和{% label success@next %}支持的一些语法特性。

## 基础的Markdown语法

[官方网站](https://daringfireball.net/projects/markdown/basics)
[在线demo](https://markdown-it.github.io/)

### 标题

一共是两种语法，我一般就使用{% label primary@# %}

- **Setext-style** - 标示样式

```markdown
A First Level Header
====================

A Second Level Header
---------------------
```

- __atx-style__ - #样式

```md
# header1

## header2
```

### 段落

直接写文字就可以了，如果要换行就空一行呗。

好吧，那我换一行开始，随便你吧。

如果处理换行，输入两个或以上的空格就可以了。

### 引用

```md
> This is a blockquote.
```
> This is a blockquote.
### 短语加强
```md
Some of these words *are emphasized*.
Some of these words _are emphasized also_.
Use two asterisks for **strong emphasis**.
Or, if you prefer, __use two underscores instead__.
```
*italic*   **bold**
_italic_   __bold__

### 列表
列表可以分成{%label info@无序%}和{%label info@有序%}。

无序列表有三种`*`,`+`,`-`

```md
*   Candy.
*   Gum.
*   Booze.

+   Candy.
+   Gum.
+   Booze.

-   Candy.
-   Gum.
-   Booze.
```

有序列表

```md
1.  Red
2.  Green
3.  Blue
```

如果列表中加入p元素，看下面

*   A list item.

    With multiple paragraphs.

*   Another item in the list.

### 链接
链接有两种样式，{%label info@内联式 %}和{%label info@引用式 %}

- 内联式

```md
An [example](http://url.com/ "Title")
```

- 引用式

```md
An [example][id]. Then, anywhereelse in the doc, define the link:

[id]: http://example.com/  "Title"
```

### 图片
图片有两种样式，一种是内联式，一种是引用的。我一般就使用内联式

1. 内联式

```md
![alt text](/path/to/img.jpg "Title")
```

2. 引用的

```md
![alt text][id]

[id]: /path/to/img.jpg "Title"
```

### 代码块

四个空格或者一个tab

This is a normal paragraph:

    This is a code block.

Here is an example of AppleScript:

    tell application "Foo"
        beep
    end tell

### 代码

使用反引号，或者多个的到引号

```md
Use the `printf()` function.
```

Use the `printf()` function.

```java
System.out.println("Hello Word!!!");
```

### 水平线

水平线就三种

---

* * *

- - - -

<!-- more -->

## Hexo中的语法

Hexo提供了{% label info@标签插件 %}，可扩展Markdown语法。详见[官方文档](https://hexo.io/zh-cn/docs/tag-plugins)

### 引用块

```md
{% blockquote [author[, source]] [link] [source_link_title] %}
content
{% endblockquote %}
```

{% blockquote @DevDocs https://twitter.com/devdocs/status/356095192085962752 %}
NEW: DevDocs now comes with syntax highlighting. http://devdocs.io
{% endblockquote %}

### 代码块

```md
{% codeblock [title] [lang:language] [url] [link text] [additional options] %}
code snippet
{% endcodeblock %}
```

{% codeblock title lang:javascript line_number:false %}
alert('Hello World!');
{% endcodeblock %}

{% codeblock _.compact http://underscorejs.org/#compact Underscore.js mark:1-2 %}
_.compact([0, 1, false, 2, '', 3]);
=> [1, 2, 3]
{% endcodeblock %}

```javascript title https://www.1haitao.com 海淘一号
alert('Hello World!');
alert('Hello World!');
```

### 链接

{% link 百度一下 https://www.baidu.com %}

```md
{% link text url [external] [title] %}
```

### 图片
可以指定图片的高度和宽度

```md
{% img [class names] /path/to/image [width] [height] '"title text" "alt text"' %}
```

{% img /images/apple-touch-icon.png 50 50 '"我的头像" "我的头像"' %}

### 引用文章
引用其他文章的连接，这个用起来还是很方便

```md
{% post_path filename %}
{% post_link filename [title] [escape] %}
```

{% post_link about-my-blog '通往文章的链接' %}

### 文章和摘要
在文章中使用 `<!-- more -->`，在这之前的文字被视为摘要，首页只出现这部分文字

## Next中支持的语法
Next在hexo的基础上又提供了一些额外的{% label info@标签插件 %}

### 居中的引用

{% cq %}Elegant in code, simple in core{% endcq %}

### Note

```md note.js
{% note [class] [no-icon] %}
Any content (support inline tags too.io).
{% endnote %}
[class]   : default | primary | success | info | warning | danger.
[no-icon] : 不适用图标
```

{% note info no-icon %}
#### No icon note
Note **without** icon: `note info no-icon`

note info, note info, note info
note info, note info, note info
note info, note info, note info
{% endnote %}

### Tabs

```md tabs.js
{% tabs Unique name, [index] %}
<!-- tab [Tab caption] [@icon] -->
Any content (support inline tags too).
<!-- endtab -->
{% endtabs %}
Unique name   : 唯一名称
[index]       : -1代表没有选中
[@icon]       : FontAwesome icon name (without 'fa-' at the begining)
```

{% tabs Fourth unique name %}
<!-- tab Solution 1 -->
**This is Tab 1.**
<!-- endtab -->

<!-- tab Solution 2 -->
**This is Tab 2.**
<!-- endtab -->

<!-- tab Solution 3 -->
**This is Tab 3.**
<!-- endtab -->
{% endtabs %}

### 标签

```md label.js
{% label [class]@Text %}
[class] : default | primary | success | info | warning | danger
```

{% label @ipsum %} {% label primary@dolor sit %} {% label success@adipiscing elit %} {% label info@do eiusmod %}

*{% label warning @ad %}* **{% label danger@nostrud %}**

~~{% label default @velit %}~~ <mark>esse</mark>

### 视频

{% video https://image.hualihai.cn/blog/843d7115206e9cb82b0f7d19a66a26.m4v %}

### 按钮

```md button.js
{% button url, text, icon [class], [title] %}
<!-- Tag Alias -->
{% btn url, text, icon [class], [title] %}
icon    : FontAwesome icon name (without 'fa-' at the begining)
[class] : FontAwesome class(es): fa-fw | fa-lg | fa-2x | fa-3x | fa-4x | fa-5x
[title] : Tooltip at mouseover
```

{% btn https://github.com/theme-next/hexo-theme-next, NexT, github fa-fw fa-lg, NexT source code %}

### 图片组

```md group-pictures.js
{% grouppicture [group]-[layout] %}{% endgrouppicture %}
{% gp [group]-[layout] %}{% endgp %}
[group]  : 总的图片数量
[layout] : 展示的图片数量
```

{% grouppicture 4-2 %}
  ![](/images/apple-touch-icon.png)
  ![](/images/apple-touch-icon.png)
  ![](/images/apple-touch-icon.png)
  ![](/images/apple-touch-icon.png)
{% endgrouppicture %}

### 表情
我是直接从[这个网站](https://emojipedia.org/people/)上拷贝表情，然后粘贴到文件里面。
😀😃😄

## 总结
这里我列举的是我使用到的一些标签插件，还有一些用不到我就没写，大家可以看看官网文档。

写文章一般的Markdown语法就够了，然后在使用标签插件，基本上就满足了我们日常的写作需求，要注意排版等  
还有就是一些特殊符号要注意，要注意转义!!!!