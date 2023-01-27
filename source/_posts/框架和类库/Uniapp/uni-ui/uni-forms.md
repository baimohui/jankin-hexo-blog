---
title: uni-forms 表单校验梳理
categories: 
- uniapp
tags:
- uniapp
- Vue
---

[参考文档](https://zh.uniapp.dcloud.io/component/uniui/uni-forms.html#%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8)
### 一、前置条件
`uni-forms` 需要绑定 `model` 属性，值为表单的 key/value 组成的对象。`uni-form-item` 需要设置 `name` 属性为当前字段名，字段为 String|Array 类型。
```html
<uni-forms :modelValue="formData">
	<uni-forms-item label="姓名" name="name">
		<uni-easyinput type="text" v-model="formData.name" placeholder="请输入姓名" />
	</uni-forms-item>
	<uni-forms-item required :name="['data','hobby']" label="兴趣爱好">
		<uni-data-checkbox multiple v-model="formData.data.hobby" :localdata="hobby"/>
	</uni-forms-item>
</uni-forms>
```

<!-- more -->

### 二、rules 属性
通过 rules 属性传入约定的校验规则。
```html
<template>
  <uni-forms ref="form" :rules="rules">
    <uni-forms-item label="姓名" name="name">
      <uni-easyinput type="text" v-model="formData.name" placeholder="请输入姓名" />
    </uni-forms-item>
    <uni-forms-item label="昵称" name="nickname">
      <uni-easyinput type="text" v-model="formData.nickname" placeholder="请输入昵称" />
    </uni-forms-item>
  </uni-forms>
</template>

<script>
  export default {
    data() {
      return {
        rules: {
          name: {
            rules: [
              // 校验 name 不能为空
              {
                required: true,
                errorMessage: '请填写姓名',
              },
              // 对 name 字段进行长度验证
              {
                minLength: 3,
                maxLength: 5,
                errorMessage: '{label}长度在 {minLength} 到 {maxLength} 个字符',
              },
              // 自定义校验规则
              {
                validateFunction: function(rule, value, data, callback){
                  if (value === data.nickname) {
                    callback('名字与昵称不能相同')
                  }
                  return true
                }
              }
            ],
            // 当前表单域的字段中文名，可不填写
            label:'姓名',
            validateTrigger:'submit'
          }
        }
      }
    }
  }
</script>
```
