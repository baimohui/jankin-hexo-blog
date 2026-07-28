---
title: AntV 可视化方案详解
categories: 
- 框架和类库
tags:
- AntV
- G2
- 数据可视化
- 图表
- 蚂蚁集团
---

## 一、AntV 是什么

AntV 是蚂蚁集团（Ant Group）开源的数据可视化解决方案体系，不是单一图表库。它涵盖了统计图表、关系图、地理可视化、移动端图表、多维表格等多个领域。<!--more-->

### AntV 产品矩阵

| 库 | 定位 | 适用场景 | GitHub Stars |
|----|------|----------|-------------|
| **G2** | 统计图表——图形语法（Grammar of Graphics） | 折线图、柱状图、散点图等通用图表 | ~12k |
| **G6** | 图可视化（Graph Visualization） | 关系图、拓扑图、流程图、树图 | ~11k |
| **L7** | 地理空间可视化 | 地图、热力图、路径图、3D 地图 | ~3.5k |
| **F2** | 移动端图表 | 手机 H5 上的轻量图表 | ~7.8k |
| **S2** | 多维交叉表格 | 透视表、分析表格、电子表格 | ~1.3k |

### AntV vs ECharts

| 维度 | AntV (G2) | ECharts |
|------|-----------|---------|
| 理念 | 图形语法——数据映射到图形属性 | 配置项——通过 option 描述图表 |
| 学习曲线 | 较高（需理解语法概念） | 低（配置式） |
| 定制性 | 极高（语法灵活组合） | 中等（配置项覆盖大部分场景） |
| 动画 | 声明式，由语法驱动 | 配置式 |
| TypeScript | 原生支持 | 支持 |
| 移动端 | F2 专门优化 | ECharts 5 已优化 |
| 包体积 | 按需引用（G2 ~500KB） | ~1MB（完整包） |

## 二、G2——统计图表

G2 基于**图形语法**（Grammar of Graphics）设计。核心思想是用数据字段映射到视觉通道（位置、颜色、大小、形状等）。

### 2.1 快速开始

```bash
npm install @antv/g2
```

```js
import { Chart } from '@antv/g2';

const chart = new Chart({
  container: 'container',
  width: 800,
  height: 400,
});

// 声明数据
chart.data([
  { year: '2020', sales: 120 },
  { year: '2021', sales: 200 },
  { year: '2022', sales: 150 },
  { year: '2023', sales: 80 },
  { year: '2024', sales: 180 },
]);

// 声明图形语法
chart.line()
  .encode('x', 'year')
  .encode('y', 'sales')
  .encode('color', 'year');

chart.render();
```

### 2.2 核心概念

**数据（Data）** → **图形语法（标记 + 编码）** → **渲染**

```text
G2 的绘图过程：
  1. chart.data(data)              → 输入数据
  2. chart.interval() / line()     → 选择图形标记（柱状/折线/散点等）
  3. .encode('x', '字段')          → 编码：数据字段映射到视觉通道
  4. .scale('x', { type: 'time' }) → 比例尺：控制映射规则
  5. .style({ fill: '#409eff' })   → 样式
  6. chart.render()                 → 渲染
```

**标记（Mark）**：图形的基本类型

```js
chart.interval()        // 柱状图/条形图
chart.line()            // 折线图
chart.point()           // 散点图
chart.area()            // 面积图
chart.polygon()         // 多边形（热力图）
chart.cell()            // 单元格（矩阵图）
chart.text()            // 文本标记
chart.image()           // 图片标记
```

**编码（Encode）**：数据字段 → 视觉通道

```js
.encode('x', 'year')        // x 轴位置
.encode('y', 'sales')       // y 轴位置
.encode('color', 'type')    // 颜色
.encode('size', 'count')    // 大小
.encode('shape', 'type')    // 形状
.encode('opacity', 'value') // 透明度
```

### 2.3 常见图表

```js
// 柱状图
chart.interval()
  .encode('x', 'city')
  .encode('y', 'value');

// 分组柱状图
chart.interval()
  .encode('x', 'city')
  .encode('y', 'value')
  .encode('color', 'type')
  .transform({ type: 'dodgeX' });   // 分组排列

// 堆叠柱状图
chart.interval()
  .encode('x', 'city')
  .encode('y', 'value')
  .encode('color', 'type')
  .transform({ type: 'stackY' });   // 堆叠

// 折线图 + 点
chart.line().encode('x', 'date').encode('y', 'price');
chart.point().encode('x', 'date').encode('y', 'price');

// 饼图
chart.pie()
  .encode('color', 'type')
  .encode('angle', 'value')
  .style({ stroke: '#fff', lineWidth: 2 });

// 散点图
chart.point()
  .encode('x', 'height')
  .encode('y', 'weight')
  .encode('color', 'gender')
  .encode('shape', 'point');
```

### 2.4 比例尺（Scale）

比例尺控制数据到视觉通道的映射方式：

```js
chart.scale('sales', {
  type: 'linear',         // 线性（默认）
  domain: [0, 200],       // 数据域
  range: [0, 400],        // 输出范围（像素）
});

chart.scale('date', {
  type: 'time',           // 时间比例尺
});

chart.scale('city', {
  type: 'band',           // 离散比例尺（柱状图 x 轴）
});

chart.scale('value', {
  type: 'log',            // 对数比例尺
});
```

### 2.5 交互

```js
// 内置交互
chart.interaction('tooltip');           // 提示框
chart.interaction('brush', {            // 框选
  brushType: 'x',                       // x 方向框选
});
chart.interaction('legend-highlight');  // 图例高亮
chart.interaction('drag-move');         // 拖拽平移
chart.interaction('scrollbar');         // 滚动条

// 事件监听
chart.on('element:click', (e) => {
  console.log(e.data.data);  // 点击的数据
});
chart.on('element:pointerenter', (e) => {});
```

### 2.6 多视图（Facet）

```js
// 分面：按 type 拆分为多个子图
chart.facetRect()
  .encode('x', 'type')
  .data(data)
  .children((child) => {
    child.interval()
      .encode('x', 'city')
      .encode('y', 'value');
  });
```

### 2.7 动画

```js
chart.interval()
  .encode('x', 'city')
  .encode('y', 'value')
  .animate('enter', {
    type: 'fadeIn',    // 入场动画
    duration: 300,
  })
  .animate('update', {
    type: 'morphing',  // 更新动画
  });
```

## 三、G6——图可视化

G6 专注于关系网络的可视化，适用于流程图、拓扑图、社交网络等。

### 3.1 快速开始

```bash
npm install @antv/g6
```

```js
import G6 from '@antv/g6';

const data = {
  nodes: [
    { id: 'node1', label: '用户 A' },
    { id: 'node2', label: '用户 B' },
    { id: 'node3', label: '用户 C' },
  ],
  edges: [
    { source: 'node1', target: 'node2' },
    { source: 'node2', target: 'node3' },
  ],
};

const graph = new G6.Graph({
  container: 'container',
  width: 800,
  height: 500,
  layout: {
    type: 'force',       // 力导向布局
  },
  defaultNode: {
    type: 'circle',
    size: 40,
  },
});

graph.data(data);
graph.render();
```

### 3.2 常见布局

```js
/* 布局类型 */
force       // 力导向（社交网络）
dagre       // 层次（流程图、DAG）
circular    // 环形
radial      // 辐射状
concentric  // 同心圆
grid        // 网格
mds         // 高维降维
```

### 3.3 G6 应用场景

```text
├── 流程图（dagre 布局）
├── 社交网络关系图（force 布局）
├── 组织架构树（dendrogram / compactBox）
├── 知识图谱（force + 自定义节点）
├── 网络拓扑图（circular / radial）
└── 依赖关系分析（dagre + 层级样式）
```

## 四、L7——地理可视化

```bash
npm install @antv/l7
```

```js
import { Scene, HeatmapLayer } from '@antv/l7';
import { GaodeMap } from '@antv/l7-maps';

const scene = new Scene({
  id: 'map',
  map: new GaodeMap({ style: 'light', center: [116.4, 39.9], zoom: 10 }),
});

// 热力图
const layer = new HeatmapLayer({})
  .source(data, { parser: { type: 'json', x: 'lng', y: 'lat' } })
  .style({ intensity: 3, radius: 20 });
scene.addLayer(layer);
```

## 五、AntV 与 React / Vue 集成

### React

```jsx
import { useEffect, useRef } from 'react';
import { Chart } from '@antv/g2';

function LineChart({ data }) {
  const container = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    if (!chartRef.current) {
      chartRef.current = new Chart({ container: container.current, autoFit: true });
    }
    const chart = chartRef.current;
    chart.data(data);
    chart.clear();  // 清除旧图形
    chart.line().encode('x', 'date').encode('y', 'value');
    chart.render();

    return () => chart.destroy();
  }, [data]);

  return <div ref={container} style={{ height: 400 }} />;
}
```

### Vue 3

```vue
<template>
  <div ref="container" style="height: 400px" />
</template>
<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';
import { Chart } from '@antv/g2';

const props = defineProps({ data: Array });
const container = ref(null);
let chart = null;

onMounted(() => {
  chart = new Chart({ container: container.value, autoFit: true });
  renderChart();
});

watch(() => props.data, () => {
  chart && renderChart();
}, { deep: true });

function renderChart() {
  chart.data(props.data);
  chart.clear();
  chart.line().encode('x', 'date').encode('y', 'value');
  chart.render();
}

onUnmounted(() => chart?.destroy());
</script>
```

## 六、面试题

### Q1: AntV G2 和 ECharts 的核心区别

```text
G2 基于图形语法——数据通过 encode 映射到视觉通道（x/y/color/size），
用户组合不同标记（interval/line/point）来构建图表，
灵活性极高，但学习曲线较陡。

ECharts 基于配置项——通过 option 描述图表的完整表现，
上手快，API 直觉化，适合标准图表场景。
```

### Q2: AntV 产品矩阵中什么时候用 G2 什么时候用 G6

```text
G2：统计图表（折线图、柱状图、散点图、饼图）
  ——数据以表格形式存在，关注数据分布和趋势

G6：关系网络图（流程图、拓扑图、社交网络）
  ——数据以节点和边存在，关注节点之间的连接关系
```

### Q3: G2 中 encode 和 scale 的区别

```text
encode：数据字段到视觉通道的映射声明（"把 date 映射到 x 轴"）
scale：编码的规则配置（x 轴是时间尺度还是线性尺度、值域范围等）

简单来说：
  encode 告诉你"什么数据对应什么视觉通道"，
  scale 告诉你"怎么对应"。
```
