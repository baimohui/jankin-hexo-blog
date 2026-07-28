---
title: Highcharts 详解
categories: 
- 框架和类库
tags:
- Highcharts
- 图表
- 数据可视化
- 兼容性
---

## 一、Highcharts 是什么

Highcharts 是一个纯 JavaScript 的图表库，以其极致的兼容性（支持 IE6+）、丰富的交互能力和优秀的文档著称。它是**商业软件**（非开源免费，个人和非商业项目免费，商业需授权）。<!--more-->

### Highcharts vs ECharts vs G2

| 维度 | Highcharts | ECharts | G2 |
|------|-----------|---------|-----|
| 许可证 | 商业授权（非商业免费） | Apache 2.0（免费） | MIT（免费） |
| 兼容性 | IE6+（最强） | IE9+ / 现代浏览器 | 现代浏览器 |
| 包体积 | ~380KB（核心） | ~1MB（完整包） | ~500KB（G2） |
| 学习曲线 | 低 | 低 | 中 |
| 交互 | 默认丰富（缩放/拖拽/钻取） | 内置丰富 | 需配置 |
| TypeScript | 支持 | 支持 | 原生支持 |
| 3D | 支持 | 支持 | 不支持 |
| 时间轴 | 极其完善（Highstock） | 完善 | 基础 |
| 地图 | Highmaps 单独产品 | 内置 | 需 L7 |

## 二、快速开始

```bash
npm install highcharts
```

```html
<div id="container" style="width: 800px; height: 400px;" />
```

```js
import Highcharts from 'highcharts';

Highcharts.chart('container', {
  title: { text: '销售额趋势' },
  xAxis: {
    categories: ['一月', '二月', '三月', '四月', '五月'],
  },
  yAxis: {
    title: { text: '销售额（万）' },
  },
  series: [{
    name: '2024',
    type: 'line',
    data: [120, 200, 150, 80, 180],
  }],
});
```

### CDN 方式

```html
<script src="https://cdn.jsdelivr.net/npm/highcharts@11/highcharts.js"></script>
<script>
  Highcharts.chart('container', { ... });
</script>
```

## 三、核心概念

### chart——图表配置

```js
Highcharts.chart('container', {
  chart: {
    type: 'line',                  // 默认图表类型
    zooming: { type: 'x' },        // 缩放
    backgroundColor: '#f5f5f5',     // 背景色
    plotBorderColor: '#ccc',
    spacing: [20, 20, 20, 20],    // 外边距
  },
  // ...
});
```

### series——系列

```js
series: [
  {
    name: '销售额',        // 系列名（图例显示）
    type: 'column',        // 系列类型（可覆盖 chart.type）
    data: [120, 200, 150],
    color: '#409eff',      // 系列颜色
  },
  {
    name: '利润',
    type: 'line',
    data: [30, 50, 40],
    yAxis: 1,               // 关联到右 Y 轴
  },
]
```

### 图表类型一览

```js
type: 'line'          // 折线图
type: 'column'        // 柱状图
type: 'bar'           // 条形图（横置柱状图）
type: 'pie'           // 饼图
type: 'scatter'       // 散点图
type: 'area'          // 面积图
type: 'spline'        // 平滑曲线
type: 'areaspline'    // 平滑面积图
type: 'heatmap'       // 热力图
type: 'treemap'       // 矩形树图
type: 'boxplot'       // 箱线图
type: 'waterfall'     // 瀑布图
type: 'funnel'        // 漏斗图
type: 'gauge'         // 仪表盘
type: 'solidgauge'    // 实心仪表盘
```

## 四、常用图表

### 柱状图

```js
Highcharts.chart('container', {
  chart: { type: 'column' },
  title: { text: '月销售额' },
  xAxis: { categories: ['一月', '二月', '三月'] },
  yAxis: { title: { text: '金额' } },
  plotOptions: {
    column: {
      borderRadius: 4,
      dataLabels: { enabled: true, format: '{y}万' },
    },
  },
  series: [
    { name: '2023', data: [80, 110, 90] },
    { name: '2024', data: [120, 200, 150] },
  ],
});
```

### 饼图

```js
Highcharts.chart('container', {
  chart: { type: 'pie' },
  title: { text: '市场份额' },
  tooltip: { pointFormat: '{series.name}: {point.percentage:.1f}%' },
  plotOptions: {
    pie: {
      allowPointSelect: true,
      cursor: 'pointer',
      dataLabels: {
        enabled: true,
        format: '<b>{point.name}</b>: {point.percentage:.1f}%',
      },
    },
  },
  series: [{
    name: '份额',
    data: [
      { name: 'Chrome', y: 65 },
      { name: 'Safari', y: 20, sliced: true },
      { name: 'Firefox', y: 10 },
      { name: '其他', y: 5 },
    ],
  }],
});
```

### 时间序列

```js
Highcharts.chart('container', {
  xAxis: {
    type: 'datetime',                          // 时间轴
    labels: { format: '{value:%Y-%m}' },
  },
  series: [{
    name: '股票价格',
    data: [
      [Date.UTC(2024, 0, 1), 100],
      [Date.UTC(2024, 1, 1), 120],
      [Date.UTC(2024, 2, 1), 110],
      [Date.UTC(2024, 3, 1), 150],
    ],
    tooltip: {
      xDateFormat: '%Y-%m-%d',
    },
  }],
});
```

## 五、交互与事件

```js
// 点击事件
Highcharts.chart('container', {
  plotOptions: {
    series: {
      cursor: 'pointer',
      point: {
        events: {
          click: function() {
            alert(`点击了: ${this.category} = ${this.y}`);
          },
        },
      },
    },
  },
});

// 图表级事件
const chart = Highcharts.chart('container', { ... });
chart.series[0].points[0].update({ y: 200 });  // 更新数据点
chart.addSeries({ name: '新增', data: [1, 2, 3] }); // 动态添加系列
chart.removeSeries(0);                            // 移除系列
```

## 六、钻取（Drilldown）

钻取是 Highcharts 的特色功能——点击图表的下钻到更细粒度的数据：

```js
Highcharts.chart('container', {
  chart: { type: 'column' },
  title: { text: '区域销售额' },
  xAxis: { type: 'category' },
  series: [{
    name: '2024',
    data: [
      { name: '华北', y: 500, drilldown: 'north' },
      { name: '华东', y: 800, drilldown: 'east' },
    ],
  }],
  drilldown: {
    series: [
      {
        id: 'north',
        name: '华北',
        data: [
          ['北京', 200],
          ['天津', 150],
          ['河北', 150],
        ],
      },
      {
        id: 'east',
        name: '华东',
        data: [
          ['上海', 300],
          ['杭州', 250],
          ['南京', 250],
        ],
      },
    ],
  },
});
```

## 七、Highstock——金融时间序列

Highstock 是 Highcharts 专门用于金融数据可视化的扩展：

```js
import Highstock from 'highcharts/highstock';

Highcharts.stockChart('container', {
  rangeSelector: {
    selected: 1,
    buttons: [
      { type: 'month', count: 1, text: '1月' },
      { type: 'month', count: 3, text: '3月' },
      { type: 'year', count: 1, text: '1年' },
      { type: 'all', text: '全部' },
    ],
  },
  series: [{
    name: '股价',
    type: 'candlestick',             // K 线图
    data: [
      [Date.UTC(2024, 0, 1), 100, 105, 98, 102],  // [时间, 开盘, 最高, 最低, 收盘]
    ],
  }],
});
```

Highstock 的特性：

```text
├── rangeSelector：时间段选择器
├── navigator：底部缩略图导航器
├── scrollbar：滚动条
├── K 线图（candlestick / ohlc）
├── 大数据量优化（dataGrouping 数据聚合）
└── 技术指标（SMA、EMA、MACD、RSI 等内置）
```

## 八、Highmaps——地图

```js
import Highcharts from 'highcharts';
import mapData from '@highcharts/map-collection/countries/cn/cn-all.geo.json';
import Highmaps from 'highcharts/modules/map';

Highmaps(Highcharts);

Highcharts.mapChart('container', {
  chart: { map: mapData },
  title: { text: '各省销售额' },
  colorAxis: { min: 0 },
  series: [{
    data: [
      ['cn-sh', 300],
      ['cn-bj', 250],
      ['cn-gd', 400],
    ],
    joinBy: ['hc-key', 0],   // GeoJSON 字段与数据匹配
    name: '销售额（万）',
    states: {
      hover: { color: '#409eff' },
    },
  }],
});
```

## 九、自适应

```js
// 默认支持容器尺寸变化自适应
// 但需要设置宽度或 chart.width

const chart = Highcharts.chart('container', {
  chart: {
    // 方式一：百分比宽度
    width: null,                    // 跟随父容器
    // 方式二：响应式配置
    reflow: true,                   // 默认 true，resize 时自动重绘
  },
});

// 手动触发
window.addEventListener('resize', () => chart.reflow());
```

## 十、主题与自定义

```js
// 内置主题
import '@highcharts/themes/dark-unica';

// 自定义主题
Highcharts.setOptions({
  colors: ['#409eff', '#67c23a', '#e6a23c', '#f56c6c'],
  chart: {
    backgroundColor: '#f5f7fa',
    style: { fontFamily: 'Microsoft YaHei' },
  },
  title: { style: { color: '#333', fontSize: '18px' } },
});
```

## 十一、常见问题

### Q1: Highcharts 免费吗

```text
非开源免费。
个人项目、非商业用途、学校/非营利组织可以免费使用。
商业用途需要购买授权（开发者授权 ~$590/年起）。
详见 highcharts.com/license。
```

### Q2: Highcharts vs ECharts 怎么选

```text
选 Highcharts：
  ├── 需要兼容老旧浏览器（IE6+）
  ├── 需要 Highstock 的金融时间序列功能
  ├── 预算允许购买商业授权
  └── 需要丰富的默认交互（钻取、缩放内置）

选 ECharts：
  ├── 免费开源无顾虑
  ├── 需要内置中国地图
  ├── 国内使用为主（中文文档和社区更好）
  └── 大数据量场景性能更优
```

### Q3: Highcharts 的 dataLabels 如何格式化

```js
dataLabels: {
  enabled: true,
  format: '{y} 万',              // {y} 数据值、{x} 类别、{name} 点名称
  formatter: function() {         // 自定义格式化函数
    return this.y > 100 ? '高: ' + this.y : this.y;
  },
}
```

### Q4: Highcharts 如何导出图片

```js
// 内置导出按钮（需引入 exporting 模块）
import Highcharts from 'highcharts';
import exportingModule from 'highcharts/modules/exporting';
exportingModule(Highcharts);

// 配置导出按钮
Highcharts.chart('container', {
  exporting: {
    enabled: true,
    buttons: {
      contextButton: {
        menuItems: ['downloadPNG', 'downloadJPEG', 'downloadPDF', 'downloadSVG'],
      },
    },
  },
});

// 代码触发导出
chart.exportChart({ type: 'image/png', filename: 'chart' });
```
