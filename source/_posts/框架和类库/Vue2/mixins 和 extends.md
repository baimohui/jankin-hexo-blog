---
title: Vue mixins 和 extends 区别
categories: 
- Vue
tags:
- Vue
- mixins
- extends
---

混合 mixins 和继承 extends

看看官方文档怎么写的，其实两个都可以理解为继承，mixins 接收对象数组（可理解为多继承），extends 接收的是对象或函数（可理解为单继承）。

<!--more-->

继承钩子函数

```js
const extend = {
    created () {
        console.log('extends created')
    }
}

const mixin1 = {
    created () {
        console.log('mixin1 created')
    }
}

const mixin2 = {
    created () {
        console.log('mixin2 created')
    }
}

export default {
    extends: extend,
    mixins: [mixin1, mixin2],
    name: 'app',
    created () {
        console.log('created')
    }
}
```

控制台输出：

```
extends created
mixin1 created
mixin2 created
created
```

结论：优先调用 mixins 和 extends 继承的父类，extends 触发的优先级更高，相对于是队列
push(extend, mixin1, minxin2, 本身的钩子函数)
经过测试，watch 的值 继承规则一样。
继承 methods

```js
const extend = {
    data () {
        return {
            name: 'extend name'
        }
    }
}
const mixin1 = {
    data () {
        return {
            name: 'mixin1 name'
        }
    }
}
const mixin2 = {
    data () {
        return {
            name: 'mixin2 name'
        }
    }
}
// name = 'name'
export default {
    mixins: [mixin1, mixin2],
    extends: extend,
    name: 'app',
    data () {
        return {
            name: 'name'
        }
    }
}
// 只写出子类，name = 'mixin2 name'，extends 优先级高会被 mixins 覆盖
export default {
    mixins: [mixin1, mixin2],
    extends: extend,
    name: 'app'
}
// 只写出子类，name = 'mixin1 name'，mixins 后面继承会覆盖前面的
export default {
    mixins: [mixin2, mixin1],
    extends: extend,
    name: 'app'
}
```



结论：子类再次声明，data 中的变量都会被重写，以子类的为准。
如果子类不声明，data 中的变量将会最后继承的父类为准。
经过测试，props 中属性、methods 中的方法 和 computed 的值 继承规则一样。
下面单独介绍下 mixins、extends、extend

mixins

调用方式：mixins: [mixin1, mixin2]

是对父组件的扩充，包括 methods、components、directive 等。。。

触发钩子函数时，先调用 mixins 的函数，再调用父组件的函数。

虽然也能在创建 mixin 时添加 data、template 属性，但当父组件也拥有此属性时以父为准，从这一点也能看出制作者的用心（扩充）。

data、methods 内函数、components 和 directives 等键值对格式的对象均以父组件/实例为准

extends

调用方式：extends: CompA

同样是对父组件的扩充，与 mixins 类似，但优先级均次于父组件

extend

扩展组件的构造器

当我们调用 vue.component('a', {...}) 时自动调用

值得注意的是 extend 内的 data 为一个函数
