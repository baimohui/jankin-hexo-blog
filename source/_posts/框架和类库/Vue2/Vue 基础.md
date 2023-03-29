---
title: Vue 基础
categories: 
- Vue
tags:
- Vue
---

## （一）MVVM 模式

### 1. MVVM 是什么

MVVM 模式，顾名思义即 Model-View-ViewModel 模式。它萌芽于 2005 年微软推出的基于 Windows 的用户界面框架 WPF，前端最早的 MVVM 框架 knockout 在 2010 年发布。 <!-- more -->

- **Model 层**：对应数据层的域模型，它主要做域模型的同步。通过 Ajax/fetch 等 API 完成客户端和服务端业务 Model 的同步。在层间关系里，它主要用于抽象出 ViewModel 中视图的 Model。 
- **View 层**：作为视图模板存在，在 MVVM 里，整个 View 是⼀个动态模板。除了定义结构、布局外，它展示的是 ViewModel 层的数据和状态。View 层不负责处理状态，View 层做的是数据绑定的声明、指令的声明、事件绑定的声明。
- **ViewModel 层**：把 View 需要的层数据暴露，并对 View 层的数据绑定声明、指令声明、事件绑定声明负责，也就是处理 View 层的具体业务逻辑。ViewModel 底层会做好绑定属性的监听。当 ViewModel 中数据变化，View 层会得到更新；而当 View 中声明了数据的双向绑定（通常是表单元素），框架也会监听 View 层（表单）值的变化。⼀旦值变化，View 层绑定的 ViewModel 中的数据也会得到自动更新。

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174612.png" alt="image-20210326141639536" style="zoom: 67%;" />

<!--示例：计数器的 MVVM-->

<img src="https://cdn.jsdelivr.net/gh/baimohui/FigureBed/img/20211106174619.png" alt="01-计数器的MVVM" style="zoom:80%;" />

### 2. MVVM 优越点

**优点：** 

1. 分离视图和模型，降低代码耦合，提高视图或者逻辑的重用性；

   比如 View 可以独立于 Model 变化和修改，⼀个 ViewModel 可以绑定在不同的 View 上，当 View 变化时 Model 可以不变，当 Model 变化时 View 也可以不变。你可以把⼀些视图逻辑放在⼀个 ViewModel 里面，让很多 view 重用这段视图逻辑。

2. 提高可测试性：ViewModel 的存在可以帮助开发者更好地编写测试代码；

3. 自动更新 dom：利用双向绑定，数据更新后视图自动更新，让开发者从繁琐的手动 dom 中解放。

**缺点：** 

1. Bug 很难被调试：因为使用双向绑定的模式，当你看到界面异常了，有可能是 View 的代码有 Bug，也可能是 Model 的代码有问题。数据绑定使得⼀个位置的 Bug 被快速传递到别的位置，要定位原始出问题的地方并不容易。另外，数据绑定的声明是指令式地写在 View 的模版当中的，这些内容是没办法去打断点 debug 的；
2. ⼀个大的模块中 model 也会很大，虽然使用方便且容易保证数据的⼀致性，但是长期持有，不释放内存就造成了更多内存的浪费；
3. 对于大型的图形应用程序，视图状态较多，ViewModel 的构建和维护的成本都会比较高。



## （二）VUE 生命周期

详情可点击：[https://juejin.cn/post/6874855535234170887](https://juejin.cn/post/6874855535234170887)

### 1. 创建阶段

创建阶段可以看做一个 vue 实例生命的开始，可以把这一阶段比作组件从受精卵到胚胎的过程，这个阶段 vue 组件开始初始化，vue 开始观察数据，这个阶段有 beforeCreate 和 created 两个生命周期钩子函数。

**beforeCreate**

`beforeCreate`：是 `new Vue()` 之后触发的第一个钩子，此时 data、methods、computed 以及 watch 上的数据和方法还未初始化，都不能被访问。

**created**

`created`：在实例创建完成后被立即调用，此时已完成以下的配置：数据观测 (data observer)，property 和方法的运算，watch/event 事件回调。然而，挂载阶段还没开始，即真实 DOM 还没有生成，$el property 目前尚不可用。在这里可以使用并更改数据，对数据的更改不会触发 updated 函数。

可以做什么：

- data 和 methods 都已经初始化好了，如果要**调用 methods 中的方法，或者操作 data 中的数据**，最早可以在此阶段中操作。
- **无法与 Dom 进行交互**，如果非要想，可以通过 `vm.$nextTick` 来访问 Dom。
- **异步数据的请求适合在 created 的钩子中使用**，例如数据初始化。

### 2. 挂载阶段

这个阶段是 vue 实例的出生阶段，这个阶段将实现 DOM 的挂载，这标志着我们**可以在浏览器里中看到页面**了。

**beforeMount**

`beforeMount`：发生在挂载之前，在这之前 template 模板已导入 render 渲染函数编译。此时虚拟 Dom 已经创建完成，即将开始渲染。在这一阶段也可以对数据进行更改，不会触发 updated。

**mounted**

`mounted`：在挂载完成后发生，此时真实的 Dom 挂载完毕，数据完成双向绑定，**可以访问到 Dom 节点，使用 $refs 属性对 Dom 进行操作**。

mounted 不会保证所有的子组件也都一起被挂载。如果你希望等到整个视图都渲染完毕，可以在 mounted 内部使用 `vm.$nextTick`。

```javascript
mounted: function () {
  this.$nextTick(function () {
    // Code that will run only after the
    // entire view has been rendered
  })
}
```

### 3. 运行阶段

vue 实例不可能一直保持不变，当 vue 实例中的数据发生改变时，DOM 也会发生变化。

**beforeUpdate**

`beforeUpdate`：发生在更新之前，即响应式数据发生更新，虚拟 dom 重新渲染之前被触发，在当前阶段对数据的更改不会造成重新渲染，但会再次触发当前钩子函数。

**updated**

`updated`：发生在更新完成之后，此时 Dom 已经更新。现在可以执行依赖于 DOM 的操作。然而在大多数情况下，你应该避免在此期间更改状态。如果要相应状态改变，最好使用计算属性或 watcher 取而代之。最好不要在此期间更改数据，因为这可能会导致无限循环的更新。

updated 不会保证所有的子组件也都一起被重绘。如果希望等到整个视图都重绘完毕，可以在 updated 里使用 `vm.$nextTick`。

```javascript
updated: function () {
  this.$nextTick(function () {
    // Code that will run only after the
    // entire view has been re-rendered（代码将在整个视图重新渲染后执行）
  })
}
```

### 4. 销毁阶段

vue 实例的消亡阶段。实例还可以被使用，直到 `destroyed()`，我们可以最后做一些想做的事情。

**beforeDestroy**

`beforeDestroy`：发生在实例销毁之前，在这期间实例完全可以被使用。此阶段适合进行善后收尾工作，比如清除计时器。

**destroyed**

`destroyed`：发生在实例销毁之后，这个时候只剩下了 dom 空壳。组件已被拆解，数据绑定被卸除，事件监听器被移除，所有子实例也统统被销毁。

在大多数场景中你不应该调用这个方法。最好使用 v-if 和 v-for 指令以数据驱动的方式控制子组件的生命周期。



## （三）Vue 组件通信

### 1. props / $emit

**父组件向子组件传值**

prop 只可以从上一级组件传递到下一级组件（父子组件），即所谓的单向数据流。而且 prop 只读，不可被修改，所有修改都会失效并警告。下面通过一个例子说明父组件如何向子组件传递数据：在子组件 `article.vue` 中如何获取父组件 `section.vue` 中的数据 `articles:['红楼梦', '西游记','三国演义']`

```html
<!--section 父组件-->
<template>
  <div class="section">
    <com-article :articles="articleList"></com-article>
  </div>
</template>
<script>
import comArticle from './test/article.vue'
export default {
  name: 'HelloWorld',
  components: { comArticle },
  data() {
    return {
      articleList: ['红楼梦', '西游记', '三国演义']
    }
  }
}
</script>
```

```html
<!--子组件 article.vue-->
<template>
  <div>
    <span v-for="(item, index) in articles" :key="index">{{item}}</span>
  </div>
</template>

<script>
export default {
  props: ['articles']
}
</script>
```

**子组件向父组件传值**

`$emit` 绑定一个自定义事件，当这个语句被执行时，就会将参数 arg 传递给父组件，父组件通过 v-on 监听并接收参数。在上个例子的基础上，点击页面渲染出来的 `ariticle` 的 `item`，父组件中显示在数组中的下标。

```html
// 父组件中
<template>
  <div class="section">
    <!-- 注意，这里触发的方法没有带括号 -->
    <com-article :articles="articleList" @onEmitIndex="onEmitIndex"></com-article>
    <p>{{currentIndex}}</p>
  </div>
</template>

<script>
import comArticle from './test/article.vue'
export default {
  name: 'HelloWorld',
  components: { comArticle },
  data() {
    return {
      currentIndex: -1,
      articleList: ['红楼梦', '西游记', '三国演义']
    }
  },
  methods: {
    onEmitIndex(idx) {
      this.currentIndex = idx
    }
  }
}
</script>
```

```html
<!--子组件中-->
<template>
  <div>
    <div v-for="(item, index) in articles" :key="index" @click="emitIndex(index)">{{item}}</div>
  </div>
</template>

<script>
export default {
  props: ['articles'],
  methods: {
    emitIndex(index) {
      this.$emit('onEmitIndex', index)
    }
  }
}
</script>
```

### 2. eventBus

`eventBus` 又称为事件总线，在 vue 中相当于所有组件共用的事件中心，可以向该中心注册发送事件或接收事件，所以组件都可以通知其他组件。当项目较大时，eventBus 容易导致难以维护。其使用方法如下：

**① 初始化**

```javascript
// event-bus.js

import Vue from 'vue'
export const EventBus = new Vue()
```

**② 发送事件**

假设你有两个组件：`additionNum` 和 `showNum`，这两个组件可以是兄弟组件也可以是父子组件；这里我们以兄弟组件为例：

```html
<!--父组件-->
<template>
  <div>
    <show-num-com></show-num-com>
    <addition-num-com></addition-num-com>
  </div>
</template>

<script>
import showNumCom from './showNum.vue'
import additionNumCom from './additionNum.vue'
export default {
  components: { showNumCom, additionNumCom }
}
</script>
```
```html
<!--addtionNum.vue 中发送事件-->
<template>
  <div>
    <button @click="additionHandle">+加法器</button>    
  </div>
</template>

<script>
import {EventBus} from './event-bus.js'
console.log(EventBus)
export default {
  data(){
    return{
      num:1
    }
  },

  methods:{
    additionHandle(){
      EventBus.$emit('addition', {
        num:this.num++
      })
    }
  }
}
</script>
```

**③ 接收事件**

```html
// showNum.vue 中接收事件

<template>
  <div>计算和：{{count}}</div>
</template>

<script>
import { EventBus } from './event-bus.js'
export default {
  data() {
    return {
      count: 0
    }
  },

  mounted() {
    EventBus.$on('addition', param => {
      this.count = this.count + param.num;
    })
  }
}
</script>
```

**④ 移除事件监听**

```javascript
import { eventBus } from 'event-bus.js'
EventBus.$off('addition', {})
```

### 3. Vuex

Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 解决了**多个视图依赖于同一状态**和**来自不同视图的行为需要变更同一状态**的问题，将开发者的精力聚焦于数据的更新而不是数据在组件之间的传递上。

**Vuex 各个模块**

- `state`：用于数据的存储，是 store 中的唯一数据源
- `getters`：如 vue 中的计算属性一样，基于 state 数据的二次包装，常用于数据的筛选和多个数据的相关性计算
- `mutations`：类似函数，改变 state 数据的唯一途径，且不能用于处理异步事件
- `actions`：类似于 `mutation`，用于提交 `mutation` 来改变状态，而不直接变更状态，可以包含任意异步操作
- `modules`：类似命名空间，用于项目中将各个模块的状态分开定义和操作，便于维护

```html
<!--父组件-->
<template>
  <div id="app">
    <ChildA/>
    <ChildB/>
  </div>
</template>

<script>
  import ChildA from './components/ChildA' // 导入 A 组件
  import ChildB from './components/ChildB' // 导入 B 组件

  export default {
    name: 'App',
    components: {ChildA, ChildB} // 注册 A、B 组件
  }
</script>
```

```html
<!--子组件 childA-->
<template>
  <div id="childA">
    <h1>我是 A 组件</h1>
    <button @click="transform">点我让 B 组件接收到数据</button>
    <p>因为你点了 B，所以我的信息发生了变化：{{BMessage}}</p>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        AMessage: 'Hello，B 组件，我是 A 组件'
      }
    },
    computed: {
      BMessage() {
        // 这里存储从 store 里获取的 B 组件的数据
        return this.$store.state.BMsg
      }
    },
    methods: {
      transform() {
        // 触发 receiveAMsg，将 A 组件的数据存放到 store 里去
        this.$store.commit('receiveAMsg', {
          AMsg: this.AMessage
        })
      }
    }
  }
</script>
```

```html
<!--子组件 childB-->
<template>
  <div id="childB">
    <h1>我是 B 组件</h1>
    <button @click="transform">点我让 A 组件接收到数据</button>
    <p>因为你点了 A，所以我的信息发生了变化：{{AMessage}}</p>
  </div>
</template>

<script>
  export default {
    data() {
      return {
        BMessage: 'Hello，A 组件，我是 B 组件'
      }
    },
    computed: {
      AMessage() {
        // 这里存储从 store 里获取的 A 组件的数据
        return this.$store.state.AMsg
      }
    },
    methods: {
      transform() {
        // 触发 receiveBMsg，将 B 组件的数据存放到 store 里去
        this.$store.commit('receiveBMsg', {
          BMsg: this.BMessage
        })
      }
    }
  }
</script>
```

```javascript
// Vuex 的 store.js
import Vue from 'vue'
import Vuex from 'vuex'
Vue.use(Vuex)
const state = {
  // 初始化 A 和 B 组件的数据，等待获取
  AMsg: '',
  BMsg: ''
}

const mutations = {
  receiveAMsg(state, payload) {
    // 将 A 组件的数据存放于 state
    state.AMsg = payload.AMsg
  },
  receiveBMsg(state, payload) {
    // 将 B 组件的数据存放于 state
    state.BMsg = payload.BMsg
  }
}

export default new Vuex.Store({
  state,
  mutations
})
```

### 4. ref / $refs

`ref`：如果在普通的 DOM 元素上使用，引用指向的就是 DOM 元素；如果用在子组件上，引用就指向组件实例，可以通过实例直接调用组件的方法或访问数据，我们看一个 `ref` 来访问组件的例子：

```javascript
// 子组件 A.vue
export default {
  data () {
    return {
      name: 'Vue.js'
    }
  },
  methods: {
    sayHello () {
      console.log('hello')
    }
  }
}
```

```javascript
// 父组件 app.vue
<template>
  <component-a ref="comA"></component-a>
</template>
<script>
  export default {
    mounted () {
      const comA = this.$refs.comA;
      console.log(comA.name);  // Vue.js
      comA.sayHello();  // hello
    }
  }
</script>
```



## （四）计算属性与监测

### computed 

- computed 是计算属性，也就是计算值，适用于计算比较消耗性能的计算场景
- computed 具有缓存性，computed 的值在 getter 执行后是会缓存的，只有在它依赖的属性值改变之后，下⼀次获取 computed 的值时才会重新调用对应的 getter 来计算 

### watch

- 更多的是「观察」的作用，类似于某些数据的监听回调，用于观察和回调进行后续操作 
- 无缓存性，页面重新渲染时值不变化也会执行或者本组件的值

**小结**

- 当我们要进行数值计算，而且依赖于其他数据，那么把这个数据设计为 computed 
- 如果你需要在某个数据变化时做⼀些事情，使用 watch 来观察这个数据变化



## （六）v-if  和 v-show

在切换 **v-if** 块时，Vue.js 有一个局部编译/卸载过程，因为 **v-if** 之中的模板也可能包括数据绑定或子组件。**v-if** 是真实的条件渲染，因为它会确保条件块在切换当中合适地销毁与重建条件块内的事件监听器和子组件。

**v-if** 也是惰性的：如果在初始渲染时条件为假，则什么也不做——在条件第一次变为真时才开始局部编译（编译会被缓存起来）。

相比之下，**v-show** 简单得多——元素始终被编译并保留，只是简单地基于 CSS 切换。

一般来说，**v-if** 有更高的切换消耗而 **v-show** 有更高的初始渲染消耗。因此，如果需要频繁切换 **v-show** 较好，如果在运行时条件不大可能改变 **v-if** 较好。

v-if 绝对是更消耗性能的，因为 v-if 在显示隐藏过程中有 DOM 的添加和删除，v-show 就简单多了，只是操作 css。

## （七）VUE 路由实现登录验证

在开发 webApp 的时候，考虑到用户体验，经常会把不需要调用个人数据的页面设置成游客可以访问，而当用户进入到一些需要个人数据的，例如购物车，个人中心，我的钱包等等，在进行登录的验证判断，如果判断已经登录，则显示页面，如果判断未登录，则直接跳转到登录页面提示用户登录，今天就来分享下如何使用 vue-router 的 beforEach 方法来实现这个需求。

### 实现

本篇文章默认您已经会使用`webpack`或者`vue-cli`来进行环境的搭建，并且具有一定的 vue 基础，如果您目前是一个新手，那么网上搜索一下就好，相关文章非常多，这里就不再赘述了。话不多说，直接上代码。为了方便日后代码的可维护性，我把相关方法写在了一个新建的 filter.js 文件里



![img](https://user-gold-cdn.xitu.io/2018/6/21/164218da430c958a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



接下来进入 filter.js 文件中，首先引入 vue-router：`import router from "./router";`然后我们使用`router.beforEach`方法：

```javascript
router.beforeEach((to, from, next) => {
    //根据字段判断是否路由过滤
    if (to.matched.some(record => record.meta.auth)) {
        if (getToken() !== null) {
            next()
        } else {
            //防止无限循环
            if (to.name === 'login') {
                next();
                return
            }
            next({
                path: '/login',
            });
        }
    } else {
        next()//若点击的是不需要验证的页面，则进行正常的路由跳转
    }
});
```

beforEach 其实是 vur-router 的钩子函数，可以理解为每个 router 跳转之前都会调用的一个方法，既然有 before 同理当然也有 afterEach，这个我们以后再讲。

首先来解释下 beforEach 的三个参数：

1. to：router 即将进入的路由对象。
2. from：当前导航正要离开的路由。
3. next：一个 function，一定要调用该方法来 resolve 这个钩子。执行效果依赖 next 方法的调用参数。

- `next()`: 进行管道中的下一个钩子。如果全部钩子执行完了，则导航的状态就是 confirmed（确认的）。
- `next(false)`: 中断当前的导航。如果浏览器的 URL 改变了（可能是用户手动或者浏览器后退按钮），那么 URL 地址会重置到 from 路由对应的地址。
- **next('/') 或者 next({ path: '/' }): 跳转到一个不同的地址。当前的导航被中断，然后进行一个新的导航。** 你可以向 next 传递任意位置对象，且允许设置诸如 replace: true、name: ‘home’之类的选项以及任何用在 router-link 的 to prop 或 router.push 中的选项，注意，next 可以通过 query 传递参数。
- `next(error)`: (2.4.0+) 如果传入 next 的参数是一个 Error 实例，则导航会被终止且该错误会被传递给 router.onError() 注册过的回调。

### 说明

好了，看到这里可能有些人还是没有理解，没关系，接下来我举个例子就可以明白了。
 假设我们目前有三个路由：`/home，/mine，/login`
 我们初始进入为`/home`，这时候点击跳转`/mine`，但是由于我们没有登录，所以会自动跳转到`/login`
 在以上这种情况下，
 to：代表着路由`/mine`，我们要进入的路由。
 from：代表着路由`/home`，我们将要离开的路由。
 注意，使用 beforEach 最后必须要调用`next()`，否则会报错，如果不传参数，我们就会成功进入到`/mine`，如果我们传递参数，例如`next('/login')`，那么我们在点击任何路由都会跳转到`/login`界面。
 但是我们的需求是只有点击需要进行登录验证的页面才进行拦截跳转，因此，我们需要加一些判断条件来进行路由的筛选。

```javascript
if (to.matched.some(record => record.meta.auth)) {
    if (getToken() !== null) {
        next()
    }
}
```

这里的 to 就是上面讲的参数 to，`to.matched`是一个对象数组，里面有 to 指向路由的相关信息，例如：path，name，meta 等等。
 我们用该数组调用 some() 方法根据返回值`true`或者`false`来进行判断，所以我们要在 router.js 路由配置文件中为我们需要验证登录判断跳转的路由添加一个字段来作为判断条件

```
{
path: '/mine',
name: 'mine',
component: mine,
meta:{auth:true}  //我们自己添加的字段
}
```

由于给路由添加了`meta:{auth:true}`，所以我们的`to.matched.some(record => record.meta.auth)`会返回`true`，这时我们就可以做登录判断了，我的项目是通过把 token 存入到`localstorage`来进行判断的，getToken() 是我封装的一个获取`localstorage`方法。

```javascript
if (getToken() !== null) {
    next() //若 token 不为 null，则进行路由跳转
}
```

如果没有 token，我们下一步继续进行判断，也就是最终目的，进行路由拦截，跳转到登录页

```javascript
else {
    next({
        path: '/login',
    });
}
```

但是这时候我们会遇到新的问题，打开控制台会发现路由会无限的循环并最终崩溃，这是什么原因呢？仔细阅读上文红色加粗，我们可以理解为

- `next()` 表示路由成功，直接进入 to 路由，不会再次调用 router.beforeEach()
- `next({ path: '/login', });` 表示路由拦截成功，重定向至 login，会再次调用 router.beforeEach()

也就是说 beforeEach() 必须调用 next()，否则就会出现无限循环
 next() 和 next('xxx') 是不一样的，区别就是前者不会再次调用 router.beforeEach()，后者会。而由于我们没有 token，所以在重新调用 router.beforeEach() 后，会再次进入到

```javascript
else {
    next({
        path: '/login',
    });
}
```

所以造成了无限循环，解决这个问题的方法也很简单，我们在`next({ path: '/login', });`之前增加一个判断条件

```javascript
if (to.name === 'login') {
    next();
    return
}
```

如果我们 to 的定向路由`name == 'login'`,则执行`next();`并 return 终止代码运行。

以上就是通过 router.beforEach 方法进行路由拦截了，我们不仅仅可以只做登录判断，通过这个方法可以实现很多需求，只要是有关路由跳转的都可以，在下只是抛砖引玉，如果有哪里不对的地方或者有更好的方法可以直接在评论告诉我，非常感谢。

**tips**

2018.7.2
 由于实际项目中很多界面都需要进行 token 的验证拦截，因此考虑后决定把过滤的方法设置在 axios 拦截器的`http response 拦截器`里，根据后端返回的错误码来判断是 token 过期或者无效之类的错误来进行跳转到登录页面。





