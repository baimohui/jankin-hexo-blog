---
title: uniapp 开发app日志
categories: 
- uniapp
tags:
- uniapp
- APP
- Vue
---

第一次接触 uniapp 项目，同时也是第一次开发 APP，难免会碰到很多问题。发现并解决问题的过程略有艰辛但又带点成就感。在此简要记录所遇到的一些问题和相应解决方案，与君共勉。<!--more-->

### 一、HBuilderX 篇

#### 1. HBuilderX 不能从 DCloud 中引入插件   

在 DCloud 市场上选择从 HBuilderX 上导入插件时，HBuilderX 打开后会报类似不能写入的错误。这是没有权限导致的，需要将程序以管理员身份运行。但由于插件导入时，默认会重新     开启 HBuilderX，所以给之前开启的程序设置管理员权限不能起到效果。这时应右键点击应用选择“属性”，然后点击“兼容性”，开启“以管理员身份运行此程序”选项。

#### 2. ios 真机调试

直接将苹果与电脑用 usb 线连接，HBuilderX 真机调试时并不会出现设备选项。我们首先需要在设置中打开允许 usb 连接的选项，然后在电脑安装 iTools。安装后打开，进行驱动修复后设备即可显示。





### 二、uniapp 篇

#### 1. props 值定义

uniapp 对于 prop 的定义好像要比 Vue 更严格。对于任何类型的 props，它都要求以如下方式进行定义：

```javascript
props: {
	list: {
		type: Array,
		default() {
			return []
		}
	}
}
```

必须要通过 construtor 来返回初始值，不然会报错。

#### 2. 真机调试报的 bugs

因为刚开始都是以 h5 的方式进行调试，但毕竟做的是 app，肯定需要通过真机调试来检验实际效果。因此在调试过程中，出现了许多原本 h5 调试没有发生的错误。

（1）开屏广告的定时器没有停止

查看代码发现定时器的停止代码是 `windows.clearInterval(timeout)` 。app 端不存在 windows 属性，直接改成 `clearInterval(timeout)` ，定时器就能正常停止了（ h5 也能执行）。

（2）不能获取到数据，报 `Expected URL scheme 'http' or 'https' but was 'file'`

因为请求的完整 URL 格式应为 http://xxx.yyy.net/pages/home。但如果系统把 URL 当作 file，可能是它接收到的 URL 就是 '/pages/home'（看着像文件路径），后来直接在请求的 URL 上添加了域名后就能获取到数据了。app 端好像不存在跨域问题。

（3）开屏广告页面图片不显示

用 webview 查看后发现图片高度为 0，在 DCloud 问答社区得知，只设置高度百分比在 app 端是无效的，如果想要高度百分比生效，那么其父元素必须要有一个确定的高度。于是我将父元素的高度设为 100 vh，图片终于成功显示。

（4）原生导航栏背景色不生效

查资料发现 navigationBackgroundColor 在 app 端的十六进制颜色值不支持缩写，即不能将 #000000 简写为 #000。写全即可。

#### 3. 路由跳转

- 页面不能跳转到其它页面：因为没有在 pages.json 中注册路由；
- 页面不能跳转到 tabbar 页面：因为没有将 open-type 设为 switchTab。不能使用 uni.navigateTo() 跳转到 tabbar 页面，而应选择 uni.switchTab()。



### 三、常识篇

#### 1. var _self = this 的由来

AJAX 请求成功调用的 success 函数里，this 值并不是全局对象，不能通过它来调用全局的对象或方法。所以需要在前面添加类似 `var _self = this` 语句，通过 _self 对象来充当全局对象。

#### 2. MobaXterm 连接 Linux 服务器

跟 uniapp 项目虽然没有直接联系，但还是要安利。刚来公司时，上司让我用 secureCrt 来连接 nginx 服务器，但是 secureCrt 安装很费功夫，因为它需要 liscence，找对应的破解版也是很让人头疼。后来无意中发现了 MobaXterm 这个工具，不仅完全免费，而且界面美观，操作友好，还自带编辑器。~~以及居然能在里面玩游戏。~~拒绝 secureCrt 是一种美德。

#### 3. /deep/ 样式穿透

后台传来的文章数据往往是 html 格式的，所以可以通过 v-html 来将其传入展示。但当我想要调整文章的样式时，比如将图片的宽度与屏幕宽度设为一致，或者在 p 段落之间要留有一些 padding，却发现在 style 中写好的样式没有生效。上网查询资料后发现，可以通过样式穿透来强行调整。

```scss
.p-paperpage__content {
	/deep/ img {
		width: 100%;
	}
	/deep/ p {
		margin-bottom: 60rpx;
	}
}
```

要注意的是，这里写的是原始 html 标签（img、p），而不是 uniapp 的标签（image、view）。

#### 4. JSON.parse()

明明看着是对象数组，系统却把它当成一长串字符。用 list[0] 提取到的不是数组的第一个对象，而是这串字符中的第一个字符。字符串还带有蛮多引号，最后确定是 json 数据，需要用 JSON.parse() 将其进行转换成数组。另外还有一个 JSON.stringify()，应该是用来转成  JSON 数据。

#### 5. Vue: template 中动态设置 background-image

```vue
<div v-for="(item, index) in list" :key=index>
	<div :style="{ backgroundImage: 'url(' + item.imgUrl + ')' }">
        item.content
    </div>
</div>
```

#### 6. 将本地代码 push 到远程仓库

首先要先将本地的 ssh 添加到远程仓库的 ssh 中。之后在 push 代码时，接连报了以下错误：

（1）`error: src refspec xxx does not match any`

查询资料后得知，这是因为我本地仓库的分支是 master，而远程仓库的分支是 develop，名字不一致导致出错。需要将本地仓库重命名：`git branch -m master develop` 。

（2）`Updates were rejected because the tip of your current branch is behind its remote counterpart`

因为远程仓库存在一个 readme.md 文件，而我没有将其同步到本地仓库中，需要将其 pull 下来：`git pull origin develop` 。

（3）`fatal: refusing to merge unrelated histories`

但 pull 时也发生报错。因为本地仓库和远程仓库的分支具有不同的提交历史，属于不同的版本，不能对其进行合并。但我们可以通过命令将其强制合并：`git pull origin develop --allow-unrelated-histories` 。

解决这些问题后，终于成功将本地代码 push 上去。

#### 7. Vue: 在 template 中使用自定义方法

在 template 中不能直接使用 import 导入的 js 方法。当子组件想要在 template 中通过自定义方法来动态调整 props 值，而这个方法代码又略长，不适宜直接放在 method 中时，我们可以通过其它一些手段来弥补。

（1）将自定义函数作为全局方法

就像 slice 这种全局方法可以在 template 中使用一样，我们可以将自定义函数在 main.js 文件中引入并绑定到 Vue.prototype 中作为全局方法。这样就能直接在 template 中使用。

```javascript
// main.js
import Vue from 'vue'
import {format} from '@/utils/request.js' //引入的自定义方法
Vue.prototype.format = format
```

```vue
// 子组件
<template>
	<ul>
        <li v-for="(item, index) in caseList" :key="index">
            <div class="img">
                <img :src="item.cover.slice(2, -2)" alt="" />
        	</div>
            <div class="text">{{ item.title }}</div>
            <!-- 直接使用，无需引入 -->
            <div class="time">{{ format(item.create_time) }}</div>
        </li>
	</ul>
</template>
```

（2）在 methods 中定义方法返回自定义函数

```vue
<template>
	<ul>
        <li v-for="(item, index) in caseList" :key="index">
            <div class="img">
                <img :src="item.cover.slice(2, -2)" alt="" />
        	</div>
            <div class="text">{{ item.title }}</div>
            <div class="time">{{ this.format(item.create_time) }}</div>
        </li>
	</ul>
</template>
<script>
	import {format} from "@/utils/timestamp.js"
	export default {
        methods: {
            format(timestamp) {
                return format(timestamp);
            }
        }
    }
</script>
```

#### 8. App.vue 中全局引入 iconfont

```vue
// App.vue
<style>
    @font-face {
        font-family: 'iconfont';
        src: url('@/static/fonts/iconfont.ttf')format('truetype');
    }
    .iconfont {
        font-family: "iconfont" !important;
        font-size: 40rpx;
        font-style: normal;
        display: inline-block;
    }
</style>
```

当要使用图标字体的时候，在要添加的元素上添加 iconfont 类，内容为对应图标的 unicode 编码。

```vue
<div class="iconfont">&#xe60e;</div>
```

#### 9. 盒模型

元素的 width 不包括 padding，除非将其 box-sizing 设为 border-box。

```css
.p-main {
    width: 100vw;
    padding: 0 20rpx;
    box-sizing: border-box; //不加这句，就溢出了
}
```

#### 10. 跳过 eslint 检查



#### 11. 字符串实用方法
