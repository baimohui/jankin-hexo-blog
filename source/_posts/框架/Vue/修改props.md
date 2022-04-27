---
title: Vue子组件修改父组件传递的props值属性
categories:
- Vue
tags:
- Vue
- map
- 父子组件通信
- props
- sync
- computed
- watch
---

需求：父组件通过 props 传递给子组件 `caseList` 数组，这里需要遍历该数组来展示每个 case 元素的一些属性，其中就包括创建时间 create_time 和图片地址 cover。但父组件传过来的 create_time 是一个时间戳，我们希望时间以年-月-日的形式来呈现。而 cover 的格式是 `["http:/xxxxxx.jpg"]`，直接赋给图片 src 是不行的。所以子组件需要想办法修改 props 值。

<!-- more -->

### 一、使用计算属性

在 computed 中获取到 props，并通过 map 遍历数组来修改属性。

```vue
// 子组件
<template>
	<ul>
        <li v-for="(item, index) in newCaseList" :key="index">
            <div class="img">
                <img :src="item.cover" alt="" />
        	</div>
            <div class="text">{{ item.title }}</div>
            <div class="time">{{ item.create_time }}</div>
        </li>
	</ul>
</template>

<script>
    export default {
        props: {
            caseList: {
                type: Array,
                default: [],
            },
        },
        computed: {
            newCaseList() {
                return this.caseList.map((item) => {
                    item.cover = item.cover.slice(2, -2);
                    item.create_time = this.format(item.create_time); //通过自定义format函数转换时间戳格式
                    return item;
                });
            },
        },
    };
</script>
```

这就是我在项目中所采用的做法。而除了计算属性，还可以通过监听属性或 sync 修饰符来修改 props。以下只简单说明用法。

### 二、使用监听属性

在子组件data中拷贝一份，注意引用类型需要深拷贝，根据需求可以watch监听。

```javascript
data() {
    return {
        newCaseList: this.list.slice()
    }
},
watch: {
    list(caseList) {
        this.newCaseList = caseList
    }
}
```

### 三、使用 sync 修饰符

父组件传入 props 时加入`.sync` 修饰符，子组件通过 `this.$emit('update:xxx', params);`修改。

```vue
// 父组件
<todo-list :list.sync="list" />
 
// 子组件
methodName(index) {
    this.$emit('update:list', this.newList)
}
```

### 四、直接在 template 中修改

~~来自 n 久之后的更新~~

其实完全不用这么麻烦，我们可以直接在 template 当中进行修改。如果所要使用的函数是已经全局定义引入了的（比如 slice 这种自带的方法，或像 nuxt 项目中在 nuxt.config.js 的 plugins 中导入函数作为全局插件），那么写法可以简略如下：

```html
// 子组件
<template>
	<ul>
        <li v-for="(item, index) in caseList" :key="index">
            <div class="img">
                <img :src="item.cover.slice(2, -2)" alt="" />
        	</div>
            <div class="text">{{ item.title }}</div>
            <div class="time">{{ format(item.create_time) }}</div>
        </li>
	</ul>
</template>

<script>
    export default {
        props: {
            caseList: {
                type: Array,
                default: [],
            },
        }
    }
</script>
```

可以看到，完全没有必要再创建新数组 newCaseList 了。但如果是在页面中 import 导入 js 方法，那么在 template 中是不能直接使用该方法的。但 template 中可以使用 methods 中定义的方法。在 methods 中将引入的 js 函数作为 method 的返回值，这样就能在 template 中访问到了。

```vue
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

