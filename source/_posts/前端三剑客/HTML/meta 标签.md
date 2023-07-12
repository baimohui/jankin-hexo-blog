---
title: HTML meta 标签总结与属性使用介绍
categories: 
- CSS
tags: 
- CSS
- meta
---

之前学习前端中，对 meta 标签的了解仅仅只是这一句。

```html
<meta charset="UTF-8">
```

<!--more-->

## 简介

在查阅 w3school 中，第一句话中的“元数据”就让我开始了 Google 之旅。然后很顺利的在英文版的 w3school 找到了想要的结果。（中文 w3school 说的是元信息，Google 和百度都没有相关的词条。但元数据在 Google 就有详细解释。所以这儿采用英文版 W3school 的解释。）

> The <meta> tag provides metadata about the HTML document. Metadata will not be displayed on the page, but will be machine parsable.

不难看出，其中的关键是 metadata，中文名叫元数据，是用于描述数据的数据。它不会显示在页面上，但是机器却可以识别。这么一来 meta 标签的作用方式就很好理解了。

> Meta elements are typically used to specify page description, keywords, author of the document, last modified, and other metadata.

The metadata can be used by browsers (how to display content or reload page), search engines (keywords), or other web services

这句话对 meta 标签用处的介绍，简洁明了。
翻译过来就是：meta 常用于定义页面的说明，关键字，最后修改日期，和其它的元数据。这些元数据将服务于浏览器（如何布局或重载页面），搜索引擎和其它网络服务。

## 组成

meta 标签共有两个属性，分别是 http-equiv 属性和 name 属性。

### 1. name 属性

name 属性主要用于描述网页，比如网页的关键词，叙述等。与之对应的属性值为 content，content 中的内容是对 name 填入类型的具体描述，便于搜索引擎抓取。
meta 标签中 name 属性语法格式是：

```html
<meta name="参数" content="具体的描述">。 
```

其中 name 属性共有以下几种参数。**(A-C 为常用属性)**

#### A. seo 相关
* keywords：用于告诉搜索引擎，你网页的关键字

```html
<meta name="keywords" content="Lxxyx,博客，文科生，前端">
```

* description：用于告诉搜索引擎网站的主要内容。

```html
<meta name="description" content="文科生，热爱前端与编程。目前大二，这是我的前端博客"> 
```

* title
```html
<meta name="title" property="og:title" content="如诗的青春——我的大学">
```

* image：封面
```html
<meta name="image" property="og:image" content="https://test-cdn.regoo.com/article/8f2d79c0-23ef-4630-8946-cf56d57c384d.png">
```

* 分享到推特
```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="如诗的青春——我的大学">
<meta name="twitter:image" content="https://test-cdn.regoo.com/article/8f2d79c0-23ef-4630-8946-cf56d57c384d.png">
<meta name="twitter:description" content="大学是我们每一个人梦想的殿堂，为了来到这个殿堂我们经历了风风雨雨。">
```

#### B. viewport(移动端的窗口)

说明：这个概念较为复杂，具体的 https://juejin.cn/post/6844903721697017864。
这个属性常用于设计移动端网页。在用 bootstrap,AmazeUI 等框架时候都有用过 viewport。

举例（常用范例）：

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

#### C. robots(定义搜索引擎爬虫的索引方式)

说明：robots 用来告诉爬虫哪些页面需要索引，哪些页面不需要索引。
content 的参数有 all,none,index,noindex,follow,nofollow。默认是 all。

举例：

```html
<meta name="robots" content="none"> 
```

具体参数如下：

1.none : 搜索引擎将忽略此网页，等价于 noindex，nofollow。
2.noindex : 搜索引擎不索引此网页。
3.nofollow: 搜索引擎不继续通过此网页的链接索引搜索其它的网页。
4.all : 搜索引擎将索引此网页与继续通过此网页的链接索引，等价于 index，follow。
5.index : 搜索引擎索引此网页。
6.follow : 搜索引擎继续通过此网页的链接索引搜索其它的网页。

#### D. author(作者)

说明：用于标注网页作者
举例：

```html
<meta name="author" content="Lxxyx,841380530@qq.com"> 
```

#### E. generator(网页制作软件)

说明：用于标明网页是什么软件做的
举例：(不知道能不能这样写)：

```html
<meta name="generator" content="Sublime Text3"> 
```

#### F. copyright(版权)

说明：用于标注版权信息
举例：

```abnf
<meta name="copyright" content="Lxxyx"> //代表该网站为Lxxyx个人版权所有。
```

#### G. revisit-after(搜索引擎爬虫重访时间)

说明：如果页面不是经常更新，为了减轻搜索引擎爬虫对服务器带来的压力，可以设置一个爬虫的重访时间。如果重访时间过短，爬虫将按它们定义的默认时间来访问。
举例：

```abnf
<meta name="revisit-after" content="7 days" >
```

#### H. renderer(双核浏览器渲染方式)

说明：renderer 是为双核浏览器准备的，用于指定双核浏览器默认以何种方式渲染页面。比如说 360 浏览器。
举例：

```abnf
<meta name="renderer" content="webkit"> //默认webkit内核
<meta name="renderer" content="ie-comp"> //默认IE兼容模式
<meta name="renderer" content="ie-stand"> //默认IE标准模式
```

### 2. http-equiv 属性

http-equiv 顾名思义，相当于 http 的文件头作用。

meta 标签中 http-equiv 属性语法格式是：

```html
<meta http-equiv="参数" content="具体的描述">
```

其中 http-equiv 属性主要有以下几种参数：

#### A. content-Type(设定网页字符集)(推荐使用 HTML5 的方式)

说明：用于设定网页字符集，便于浏览器解析与渲染页面
举例：

```awk
<meta http-equiv="content-Type" content="text/html;charset=utf-8">  //旧的HTML，不推荐

<meta charset="utf-8"> //HTML5设定网页字符集的方式，推荐使用UTF-8
```

#### B. X-UA-Compatible(浏览器采取何种版本渲染当前页面)

说明：用于告知浏览器以何种版本来渲染页面。（一般都设置为最新模式，在各大框架中这个设置也很常见。）
举例：

```abnf
 <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1"/> //指定IE和Chrome使用最新版本渲染当前页面
```

#### C. cache-control(指定请求和响应遵循的缓存机制)

###### 用法 1.

说明：指导浏览器如何缓存某个响应以及缓存多长时间。这一段内容我在网上找了很久，但都没有找到满意的。
最后终于在 Google Developers 中发现了我想要的答案。

举例：

```html
<meta http-equiv="cache-control" content="no-cache">
```

共有以下几种用法：

1. no-cache: 先发送请求，与服务器确认该资源是否被更改，如果未被更改，则使用缓存。
2. no-store: 不允许缓存，每次都要去服务器上，下载完整的响应。（安全措施）
3. public : 缓存所有响应，但并非必须。因为 max-age 也可以做到相同效果
4. private : 只为单个用户缓存，因此不允许任何中继进行缓存。（比如说 CDN 就不允许缓存 private 的响应）
5. maxage : 表示当前请求开始，该响应在多久内能被缓存和重用，而不去服务器重新请求。例如：max-age=60 表示响应可以再缓存和重用 60 秒。

> [参考链接：HTTP 缓存](https://link.segmentfault.com/?enc=eJ9Ao6zEU5SXCIurC6Hf3g%3D%3D.YzZy4y1vll79fX39he3hJKfjybsNETFv%2FoYLJf%2F%2FwSg6tiFt53piMUty6ptIevuzv3T3HTzrU8IvTZVFwHj%2FQ%2BTaxC5J2sV9X9bHxN%2BqJM21QpyLq%2BmD%2B3tkUPlMYts321eP3qjZ8DzYAOmTjk7WNfjrdTkYizptxXaCG6gLHqk%3D)

###### 用法 2.(禁止百度自动转码)

说明：用于禁止当前页面在移动端浏览时，被百度自动转码。虽然百度的本意是好的，但是转码效果很多时候却不尽人意。所以可以在 head 中加入例子中的那句话，就可以避免百度自动转码了。
举例：

```abnf
<meta http-equiv="Cache-Control" content="no-siteapp" />
```

#### D. expires(网页到期时间)

说明：用于设定网页的到期时间，过期后网页必须到服务器上重新传输。
举例：

```abnf
<meta http-equiv="expires" content="Sunday 26 October 2016 01:00 GMT" />
```

#### E. refresh(自动刷新并指向某页面)

说明：网页将在设定的时间内，自动刷新并调向设定的网址。
举例：

```abnf
<meta http-equiv="refresh" content="2；URL=http://www.lxxyx.win/"> //意思是2秒后跳转向我的博客
```

#### F. Set-Cookie(cookie 设定)

说明：如果网页过期。那么这个网页存在本地的 cookies 也会被自动删除。

```abnf
<meta http-equiv="Set-Cookie" content="name, date"> //格式

<meta http-equiv="Set-Cookie" content="User=Lxxyx; path=/; expires=Sunday, 10-Jan-16 10:00:00 GMT"> //具体范例
```

## 最后

暂时总结的就这么多了，meta 标签的自定义属性实在太多了。所以只去找了常用的一些，还有像`Window-target`这样已经基本被废弃的属性，我也没有添加。
一开始以为一两个小时就能学习完毕，结果没想到竟然花了五六个小时，各处查资料，推敲文字。敲击文字的时候，也感觉自己学习了非常多。比如基本的 SEO，HTTP 协议的再次理解等。