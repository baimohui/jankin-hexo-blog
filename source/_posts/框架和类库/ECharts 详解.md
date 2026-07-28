---
title: ECharts 详解
categories: 
- 框架和类库
tags:
- ECharts
- 可视化
- 图表
- 数据可视化
---

## 一、ECharts 是什么

ECharts 是百度开源（现由 Apache 基金会管理）的 JavaScript 可视化图表库，提供丰富的图表类型和强大的交互能力，适配 PC 和移动端。<!--more-->

### 与其他图表库对比

| 特性 | ECharts | Chart.js | D3.js | Highcharts |
|------|---------|----------|-------|------------|
| 上手难度 | 低 | 低 | 高 | 低 |
| 图表类型 | **50+** 种 | 8 种 | 任意（需自行组合） | 30+ 种 |
| 大数据量 | **优秀**（WebGL 加速） | 一般 | 优秀 | 一般 |
| 交互 | 内置丰富 | 基础 | 需自行实现 | 内置 |
| 中国地图 | ✅ 内置 | ❌ | ❌ | ❌ |
| 商业许可 | Apache 2.0（免费） | MIT（免费） | BSD（免费） | **付费** |
| 移动端 | 触摸交互优化 | 支持 | 需自行优化 | 支持 |

## 二、快速开始

### 安装

```bash
npm install echarts
```

### CDN

```html
<script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
```

### 渲染第一个图表

```html
<div id="chart" style="width: 600px; height: 400px;"></div>
```

```js
import * as echarts from 'echarts';

// 1. 初始化实例
const chart = echarts.init(document.getElementById('chart'));

// 2. 指定配置项
const option = {
  title: { text: '销售额趋势' },
  tooltip: {},
  xAxis: {
    data: ['周一', '周二', '周三', '周四', '周五', '周六', '周日'],
  },
  yAxis: {},
  series: [{
    name: '销售额',
    type: 'line',        // 图表类型：line / bar / pie / scatter 等
    data: [120, 200, 150, 80, 70, 110, 130],
  }],
};

// 3. 渲染
chart.setOption(option);
```

## 三、核心概念

### 实例（Instance）

`echarts.init(dom, theme?, opts?)` 创建实例。一个 DOM 节点只能创建一个实例。

```js
const chart = echarts.init(
  document.getElementById('chart'),
  'dark',                              // 主题：light / dark 或注册的自定义主题
  { renderer: 'canvas' }               // 渲染器：canvas / svg
);
```

### option（配置项）

所有图表配置集中在一个 option 对象中。核心结构：

```js
const option = {
  title: {},           // 标题
  tooltip: {},         // 提示框
  legend: {},          // 图例
  grid: {},            // 直角坐标系布局
  xAxis: {},           // x 轴
  yAxis: {},           // y 轴
  series: [],          // 系列列表（核心！每个系列是一种图表类型）
  color: [],           // 调色盘
  toolbox: {},         // 工具栏（导出图片、数据视图等）
  dataZoom: [],        // 数据区域缩放
  visualMap: {},       // 视觉映射
};
```

`setOption` 是**增量更新**的——ECharts 会自动 diff 新旧 option，只更新变化部分：

```js
// 第一次设置
chart.setOption({ series: [{ data: [1, 2, 3] }] });

// 增量更新：只修改 data，其他配置保持不变
chart.setOption({ series: [{ data: [4, 5, 6] }] });
```

### 系列（Series）

`series` 是 option 的核心，每个系列对应一种图表类型。

```js
series: [
  {
    name: '销售额',
    type: 'line',      // 系列类型
    data: [120, 200],   // 数据
    smooth: true,       // 平滑曲线
    areaStyle: {},      // 面积图
  },
  {
    name: '利润',
    type: 'bar',
    data: [30, 50],
  },
]
```

## 四、常用图表类型

### 折线图

```js
option = {
  xAxis: { type: 'category', data: ['Jan', 'Feb', 'Mar'] },
  yAxis: { type: 'value' },
  series: [{
    type: 'line',
    data: [100, 200, 150],
    smooth: true,
    areaStyle: { opacity: 0.3 },
    markLine: { data: [{ type: 'average', name: '平均值' }] },
    markPoint: {
      data: [
        { type: 'max', name: '最大值' },
        { type: 'min', name: '最小值' },
      ],
    },
  }],
};
```

### 柱状图

```js
option = {
  xAxis: { data: ['A', 'B', 'C'] },
  yAxis: {},
  series: [{
    type: 'bar',
    data: [30, 80, 45],
    itemStyle: {
      borderRadius: [4, 4, 0, 0],
      color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
        { offset: 0, color: '#409eff' },
        { offset: 1, color: '#79bbff' },
      ]),
    },
  }],
};
```

#### 多系列柱状图（分组）

```js
option = {
  xAxis: { data: ['A', 'B', 'C'] },
  yAxis: {},
  series: [
    { name: '2023', type: 'bar', data: [30, 80, 45] },
    { name: '2024', type: 'bar', data: [40, 90, 55] },
  ],
};
```

#### 堆叠柱状图

```js
option = {
  xAxis: { data: ['A', 'B', 'C'] },
  yAxis: {},
  series: [
    { name: '基础', type: 'bar', stack: 'total', data: [30, 40, 50] },
    { name: '高级', type: 'bar', stack: 'total', data: [20, 30, 25] },
  ],
};
```

### 饼图

```js
option = {
  tooltip: { trigger: 'item', formatter: '{b}: {c} ({d}%)' },
  series: [{
    type: 'pie',
    radius: ['40%', '70%'],      // 内径/外径 → 环形图
    center: ['50%', '50%'],      // 圆心位置
    data: [
      { value: 1048, name: '搜索引擎' },
      { value: 735, name: '直接访问' },
      { value: 580, name: '邮件营销' },
      { value: 484, name: '联盟广告' },
      { value: 300, name: '视频广告' },
    ],
    roseType: 'area',            // 南丁格尔玫瑰图
    label: { formatter: '{b}\n{d}%' },
    emphasis: {
      itemStyle: { shadowBlur: 10, shadowColor: 'rgba(0,0,0,.3)' },
    },
  }],
};
```

### 散点图

```js
option = {
  xAxis: {},
  yAxis: {},
  series: [{
    type: 'scatter',
    data: [
      [10, 20], [15, 35], [20, 30], [25, 45], [30, 40],
      [35, 55], [40, 50], [45, 65], [50, 60],
    ],
    symbolSize: (val) => val[1] / 5,    // 气泡图：点大小映射到数据
  }],
};
```

### 雷达图

```js
option = {
  radar: {
    indicator: [
      { name: '销售', max: 100 },
      { name: '管理', max: 100 },
      { name: '技术', max: 100 },
      { name: '客服', max: 100 },
      { name: '研发', max: 100 },
    ],
  },
  series: [{
    type: 'radar',
    data: [{ value: [90, 70, 80, 60, 95], name: '能力评估' }],
  }],
};
```

## 五、交互与事件

### 图表事件

```js
chart.on('click', (params) => {
  console.log(params.name);     // 数据项名称
  console.log(params.value);    // 数据值
  console.log(params.seriesName); // 系列名
});

// 鼠标悬浮
chart.on('mouseover', (params) => {});

// 图表区域事件
chart.on('globalout', () => {});

// 下钻示例
chart.on('click', (params) => {
  if (params.componentType === 'series') {
    fetch(`/api/detail/${params.name}`).then(res => {
      chart.setOption({
        series: [{ data: res.data }]
      });
    });
  }
});
```

### 图表的联动

多个图表实例可以通过事件实现联动：

```js
const chart1 = echarts.init(dom1);
const chart2 = echarts.init(dom2);

chart1.on('click', (params) => {
  chart2.dispatchAction({
    type: 'highlight',
    seriesIndex: 0,
    dataIndex: params.dataIndex,
  });
});
```

### 工具栏

```js
option = {
  toolbox: {
    feature: {
      saveAsImage: {},       // 导出图片
      dataView: {},          // 数据视图
      restore: {},           // 重置
      dataZoom: {},          // 区域缩放
      magicType: {           // 动态切换图表类型
        type: ['line', 'bar', 'stack'],
      },
    },
  },
};
```

## 六、响应式与自适应

### 容器尺寸变化

```js
window.addEventListener('resize', () => {
  chart.resize();
});
```

### ResizeObserver（更精准）

```js
const observer = new ResizeObserver(() => chart.resize());
observer.observe(container);
```

### 配置百分比

ECharts 支持百分比值，会按容器实际尺寸自动计算：

```js
option = {
  series: [{ center: ['50%', '50%'], radius: ['30%', '50%'] }],
  grid: { left: '10%', right: '10%', top: 60, bottom: 40 },
};
```

## 七、数据更新

### 全量替换

```js
chart.setOption({ series: [{ data: newData }] });
```

### 追加数据（实时图表）

```js
setInterval(() => {
  chart.setOption({
    series: [{ data: getLatestData() }],
  });
}, 1000);
```

### 动态添加系列

```js
chart.setOption({
  series: [
    { name: '原有', type: 'line', data: [1, 2, 3] },
    { name: '新增', type: 'line', data: [4, 5, 6] },   // 新增系列
  ],
});
```

### 加载动画

```js
chart.showLoading({
  text: '加载中...',
  maskColor: 'rgba(255,255,255,.8)',
});

// 数据就绪后
chart.hideLoading();
chart.setOption(option);
```

## 八、主题与样式

### 内置主题

```js
const chart = echarts.init(dom, 'dark');    // light / dark
```

### 自定义颜色

```js
// 全局调色盘
option = {
  color: ['#409eff', '#67c23a', '#e6a23c', '#f56c6c', '#909399'],
};
```

### 注册主题

```js
echarts.registerTheme('myTheme', {
  backgroundColor: '#f5f5f5',
  color: ['#409eff', '#67c23a', '#e6a23c'],
  textStyle: { fontFamily: 'Microsoft YaHei' },
});

const chart = echarts.init(dom, 'myTheme');
```

## 九、大数据量优化

ECharts 在数据量超过千条时提供多种优化手段：

### 采样

```js
option = {
  series: [{
    type: 'line',
    data: largeDataSet,
    sampling: 'lttb',        // Largest Triangle Three Buckets 算法
    // sampling: 'average',   // 平均采样
    // sampling: 'max',       // 最大值采样
    // sampling: 'min',       // 最小值采样
  }],
};
```

### 渐进渲染

```js
option = {
  series: [{
    type: 'scatter',
    data: hugeDataSet,
    progressive: 500,         // 每帧渲染数量
    progressiveThreshold: 3000, // 超过此值启用渐进渲染
  }],
};
```

### 关闭动画

```js
option = {
  animation: false,
};
```

## 十、与框架集成

### React

```jsx
import { useEffect, useRef } from 'react';
import * as echarts from 'echarts';

function Chart({ option, style }) {
  const container = useRef(null);
  const instance = useRef(null);

  useEffect(() => {
    instance.current = echarts.init(container.current);
    return () => instance.current?.dispose();
  }, []);

  useEffect(() => {
    instance.current?.setOption(option);
  }, [option]);

  return <div ref={container} style={style} />;
}
```

### Vue 3

```vue
<template>
  <div ref="container" :style="{ width, height }"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';
import * as echarts from 'echarts';

const props = defineProps({ option: Object, width: String, height: String });
const container = ref(null);
let chart = null;

onMounted(() => {
  chart = echarts.init(container.value);
  chart.setOption(props.option);
});

watch(() => props.option, (opt) => chart?.setOption(opt), { deep: true });

onUnmounted(() => chart?.dispose());
</script>
```

## 十一、常见问题

### Q1: 图表容器有宽高但图表显示空白

```js
// 容器可能是 display: none 或尺寸为 0，先设置容器尺寸再 init
// 或在容器可见后调用 resize
setTimeout(() => chart.resize(), 100);
```

### Q2: setOption 不生效

```js
// 可能原因：series 中的 type 缺失、data 为空数组、xAxis/yAxis 不匹配
// 用 console.log(chart.getOption()) 检查当前配置
```

### Q3: 内存泄漏

```js
// 组件卸载时必须销毁实例
chart.dispose();
```

### Q4: 图表无法导出高清图片

```js
// 方案一：使用 toolbox 的 saveAsImage
// 方案二：通过 getDataURL 生成
const url = chart.getDataURL({ type: 'png', pixelRatio: 2, backgroundColor: '#fff' });
```

### Q5: 同一个页面多个图表

每个图表必须有独立的 DOM 容器，每个容器只调用一次 `init`。多个实例之间数据独立。

## 十二、推荐学习路径

1. 掌握常见图表类型：line、bar、pie、scatter、radar
2. 理解 option 结构：title、tooltip、legend、xAxis、yAxis、series
3. 学会事件监听和交互（点击下钻、联动）
4. 掌握响应式：`resize`、百分比布局
5. 了解大数据优化：sampling、progressive
6. 与 React / Vue 集成，做好 dispose 防止内存泄漏
