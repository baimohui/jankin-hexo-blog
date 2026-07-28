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

当数据量超过千条时，ECharts 提供的优化手段直接影响图表是否能流畅交互。优化的核心思路是**减少渲染开销**和**降低重绘频率**。

### 9.1 采样（Sampling）

采样是在**数据量极大**（数万到百万级）时，在保持数据趋势的前提下减少绘图点数的策略。ECharts 在折线图中内置了多种采样算法。

```js
option = {
  series: [{
    type: 'line',
    data: largeDataSet,       // 假设 10 万条数据
    sampling: 'lttb',         // ★ 推荐：保留视觉特征最好
  }],
};
```

#### 各采样算法对比

```text
│ 算法        │ 原理                          │ 视觉保留 | 性能 | 适用场景              │
│────────────┼───────────────────────────────┼─────────┼──────┼─────────────────────│
│ lttb       │ Largest Triangle Three Buckets│ ★★★★★   │ ★★★★ │ 通用推荐，保留峰值趋势  │
│            │ 将数据分段，每段中取面积最大的 │         │      │                     │
│            │ 三角形中间点，保留转折特征     │         │      │                     │
├────────────┼───────────────────────────────┼─────────┼──────┼─────────────────────┤
│ average    │ 每段取平均值                   │ ★★★     │ ★★★★★ │ 数据波动不大时使用    │
├────────────┼───────────────────────────────┼─────────┼──────┼─────────────────────┤
│ max        │ 每段取最大值                   │ ★★      │ ★★★★★ │ 关注上限的场景（监控）│
├────────────┼───────────────────────────────┼─────────┼──────┼─────────────────────┤
│ min        │ 每段取最小值                   │ ★★      │ ★★★★★ │ 关注下限的场景（监控）│
├────────────┼───────────────────────────────┼─────────┼──────┼─────────────────────┤
│ sum        │ 每段求和（仅堆叠图使用）       │ ★★      │ ★★★★★ │ 堆叠面积图            │
└────────────┴───────────────────────────────┴─────────┴──────┴─────────────────────┘
```

**LTTB 算法原理**（是折线图默认推荐算法）：

```text
1. 将数据按比例等分为 n 段（段数 = 图表像素宽度）
2. 从左到右，每段中选一个"代表点"
3. 选择标准：该点与前一段选中的点和后一段中点构成的三角形面积最大
4. 面积最大 → 说明该点偏离趋势最远 → 保留了趋势转折特征

结果是：折线的视觉形状几乎不变，但绘制点数从 10 万锐减到几百。
```

### 9.2 渐进渲染（Progressive）

渐进渲染用于**数据量非常大**（散点图超过数万点）时，将渲染分帧完成，避免主线程长时间阻塞导致页面卡死。

```js
option = {
  series: [{
    type: 'scatter',
    data: hugeDataSet,           // 10 万个散点
    progressive: 500,            // 每帧渲染 500 个点
    progressiveThreshold: 3000,  // 超过 3000 个点时启用渐进渲染
  }],
};
```

#### 渐进渲染的工作原理

```text
关闭渐进渲染（数据量大时）：
  数据 → 一次性全部绘制 → 主线程阻塞 5 秒 → 用户看到图表
  ↑ [5 秒内页面无响应，无法滚动/点击]

开启渐进渲染：
  数据 → 绘制前 500 个 → 交给下一帧 → 再绘 500 个 → ...
  ↑ [页面始终可操作，图表逐步显示完整]
```

#### 关键参数说明

```js
option = {
  series: [{
    // progressive：每帧渲染的数据量
    progressive: 500,
    // 建议值范围：200~2000
    // 太小 → 帧数多，总渲染时间更长
    // 太大 → 单帧耗时高，失去渐进意义

    // progressiveThreshold：启用渐进渲染的阈值
    progressiveThreshold: 3000,
    // 数据量低于此值时直接一次性渲染

    // 还可配合以下参数
    progressiveChunkMode: 'mod',  // 默认 'sequential'
    // 'sequential'：从前往后逐块渲染
    // 'mod'：间隔取块，先渲染少量点展示整体分布，再补充细节
  }],
};
```

**注意**：渐进渲染开启后，如果用户启用了 tooltip 的 `trigger: 'axis'`，ECharts 需要遍历所有数据点来计算哪个点在轴附近——大数据量时可能会卡。解决方案：

```js
tooltip: {
  trigger: 'item',     // 改为 item 模式，只计算鼠标所在位置的点
  // 或限制渲染范围
}
```

### 9.3 关闭动画

动画在数据量小时提升体验，在大数据量时却会拖慢首次渲染速度。

```js
// 大数据量场景——关闭动画
option = {
  animation: false,
};

// 需要动画时——缩短动画时间
option = {
  animationDuration: 200,   // 默认 1000
  animationEasing: 'linear',
};
```

### 9.4 dataZoom 性能优化

`dataZoom` 是实现大数据"窗口浏览"的最有效手段——只渲染可视区域的数据点。

```js
option = {
  dataZoom: [{
    type: 'inside',         // 鼠标滚轮缩放
    start: 0,               // 显示前 10% 的数据
    end: 10,
    // minSpan: 1,          // 最小窗口比例（防止用户缩到太大）
  }, {
    type: 'slider',         // 底部滑块
    start: 0,
    end: 10,
  }],
};
```

使用 `dataZoom` 后，ECharts 内部只渲染 `start` 到 `end` 范围内的数据点。配合采样使用时，采样也是在裁剪后的子集上进行。

### 9.5 其他大数据量策略

```js
// 1. 使用 SVG 渲染器（某些场景更快）
const chart = echarts.init(dom, null, { renderer: 'svg' });
// Canvas 渲染器：重绘整张画布，适合频繁更新
// SVG 渲染器：DOM 节点操作，适合大量静态数据交互

// 2. 禁止可交互操作（纯展示时）
chart.setOption({
  animation: false,
  tooltip: { show: false },         // 不显示 tooltip
  legend: { show: false },          // 不显示图例
});

// 3. 利用 appendData 增量追加（WebSocket 数据流）
const chart = echarts.init(dom);
chart.setOption({
  series: [{ type: 'scatter' }],
});

// 分批追加数据（避免一次 setOption 传输大量数据）
// 注意：仅部分图表类型支持（scatter、line 等）
chart.appendData({
  seriesIndex: 0,
  data: batchData,                  // 追加而非替换
});

// 4. 开启分片渲染
// ECharts 5 内置分片渲染机制，无需手动配置
// 当 series.data.length > 10000 时自动启用
```

### 9.6 性能对比参考

```text
测试数据：10 万条散点图数据

│ 优化策略               │ 渲染耗时   │ 交互帧率   │ 首屏可见       │
│───────────────────────│───────────│───────────│────────────────│
│ 无优化                 │ 4200ms    │ 5fps      │ 等待全部渲染   │
│ 渐进渲染（progressive）│ 1800ms    │ 30fps     │ 逐步显示       │
│ 采样（lttb）           │ 120ms     │ 55fps     │ 立刻显示       │
│ dataZoom + 采样        │ 80ms      │ 58fps     │ 只渲染窗口区域 │
│ SVG 渲染器             │ 3000ms    │ 45fps     │ 需更长时间渲染 │
│ appendData 增量追加    │ 每批 50ms  │ 60fps     │ 立刻显示部分   │
```

### 9.7 大数据量场景选型建议

```text
数据量 < 1 万：
  不需要任何优化，默认配置即可

数据量 1 万 ~ 10 万（折线图）：
  开启 sampling: 'lttb'，配合 dataZoom
  数据量 ≤ 3 万时可关闭采样，仅使用 dataZoom

数据量 10 万 ~ 100 万（散点图）：
  开启 progressive + progressiveThreshold
  开启 sampling
  使用 dataZoom 限制渲染窗口
  考虑用 appendData 分批追加
  关闭动画

数据量 > 100 万：
  服务端聚合后再传给前端
  使用 ECharts GL 的 WebGL 加速渲染
  后端做空间索引（预聚合为热力图或网格）
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
