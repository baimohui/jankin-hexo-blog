---
title: WebGL 与 Three.js 详解
categories: 
- 浏览器
tags:
- 浏览器
- WebGL
- Three.js
- 3D
- 图形学
---

## 一、WebGL 是什么

WebGL（Web Graphics Library）是浏览器中实现 3D 渲染的 JavaScript API，基于 OpenGL ES 2.0/3.0 标准。它利用 GPU 进行硬件加速，能够在浏览器中渲染复杂的 3D 场景。<!--more-->

### WebGL vs Canvas 2D

| 维度 | Canvas 2D | WebGL |
|------|-----------|-------|
| 渲染方式 | CPU 渲染 | **GPU 硬件加速** |
| 维度 | 2D | 2D / **3D** |
| 图形数量 | 万级 | **百万级** |
| 着色器 | 无 | **顶点着色器 + 片元着色器** |
| 学习曲线 | 低 | 高 |
| 适用场景 | 图表、2D 游戏 | 3D 游戏、可视化、VR/AR |

### 直接使用 WebGL 还是 Three.js

**原生 WebGL** 的代码量极大——绘制一个旋转立方体需要约 200 行代码（顶点数据、着色器编译、缓冲区绑定、矩阵计算等）。Three.js 是对 WebGL 的高层次封装，将上述细节抽象为 Scene、Camera、Renderer、Mesh 等对象。

```js
// 原生 WebGL：绘制一个三角形约 80 行
// Three.js：绘制一个旋转立方体约 20 行
```

**推荐路径**：通过 Three.js 入门 3D 开发，理解 WebGL 概念（着色器、缓冲区、管线）以排查性能问题。

## 二、Three.js 核心概念

Three.js 的核心对象构成一个完整的 3D 场景：

```
Scene（场景）
  ├── Camera（相机）    → 决定了从哪个角度观察
  ├── Renderer（渲染器） → 负责将场景绘制到 Canvas
  ├── Light（光源）     → 照亮场景中的物体
  └── Mesh（网格）      → 可见物体 = Geometry + Material
       ├── Geometry（几何体）  → 形状（顶点数据）
       └── Material（材质）    → 外观（颜色、纹理、光照响应）
```

### 最小 3D 场景

```bash
npm install three
```

```js
import * as THREE from 'three';

// 1. 场景
const scene = new THREE.Scene();

// 2. 透视相机（视角 75°，宽高比，近裁剪面 0.1，远裁剪面 1000）
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 5;

// 3. 渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 4. 物体：几何体 + 材质 = 网格
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x409eff });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// 5. 光源（MeshStandardMaterial 需要光照才能显示）
const light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(5, 5, 5);
scene.add(light);

// 6. 动画循环
function animate() {
  requestAnimationFrame(animate);
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  renderer.render(scene, camera);
}
animate();
```

## 三、场景（Scene）

Scene 是所有物体的容器。可以向场景中添加或移除物体：

```js
scene.add(cube);         // 添加物体
scene.remove(cube);      // 移除物体
scene.add(light);        // 光源也是场景的一部分

// 场景背景色
scene.background = new THREE.Color(0x1a1a2e);

// 雾效（远处物体渐变消失）
scene.fog = new THREE.Fog(0x1a1a2e, 10, 50);
```

## 四、相机（Camera）

### 透视相机 vs 正交相机

```js
// 透视相机：近大远小（真实感）
const camera = new THREE.PerspectiveCamera(
  75,                              // 视野角度 FOV
  window.innerWidth / window.innerHeight,  // 宽高比
  0.1,                             // 近裁剪面（太近的物体不渲染）
  1000                             // 远裁剪面（太远的物体不渲染）
);

// 正交相机：无透视变形（用于 2D 游戏、UI）
const camera = new THREE.OrthographicCamera(
  -10, 10,   // left, right
  10, -10,   // top, bottom（注意：y 轴上下颠倒）
  0.1, 100
);
```

### 相机控制（OrbitControls）

```bash
npm install three/examples/jsm/controls/OrbitControls.js
```

```js
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;         // 启用惯性
controls.dampingFactor = 0.05;
controls.autoRotate = true;
controls.autoRotateSpeed = 2;
controls.target.set(0, 0, 0);
```

```js
// 动画循环中需调用 update
function animate() {
  controls.update();          // 处理用户交互
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## 五、几何体（Geometry）

Three.js 提供了丰富的内置几何体：

```js
const box = new THREE.BoxGeometry(1, 1, 1);              // 立方体
const sphere = new THREE.SphereGeometry(1, 32, 32);       // 球体（半径，分段数）
const cylinder = new THREE.CylinderGeometry(1, 1, 2, 32); // 圆柱体
const cone = new THREE.ConeGeometry(1, 2, 32);            // 圆锥体
const plane = new THREE.PlaneGeometry(5, 5);              // 平面
const torus = new THREE.TorusGeometry(1, 0.4, 16, 100);   // 环面
const ring = new THREE.RingGeometry(0.5, 1, 32);          // 圆环
const circle = new THREE.CircleGeometry(1, 32);           // 圆形
```

分段数越高，表面越平滑，但顶点数越多：

```js
// 低分段 → 性能好但棱角明显
const lowPoly = new THREE.SphereGeometry(1, 8, 6);

// 高分段 → 平滑但消耗大
const smooth = new THREE.SphereGeometry(1, 64, 64);
```

### 自定义几何体

```js
const geometry = new THREE.BufferGeometry();

// 顶点坐标
const vertices = new Float32Array([
  -1, -1, 0,
   1, -1, 0,
   0,  1, 0,
]);
geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));

// 顶点颜色
const colors = new Float32Array([
  1, 0, 0,   // 红
  0, 1, 0,   // 绿
  0, 0, 1,   // 蓝
]);
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

// 索引（复用顶点）
const indices = [0, 1, 2];
geometry.setIndex(indices);

// 法线（用于光照计算）
geometry.computeVertexNormals();

const material = new THREE.MeshBasicMaterial({ vertexColors: true });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

## 六、材质（Material）

| 材质 | 特点 | 需要光源 |
|------|------|---------|
| `MeshBasicMaterial` | 基础颜色/纹理，无光照响应 | ❌ |
| `MeshStandardMaterial` | PBR 材质，物理真实 | ✅ |
| `MeshPhongMaterial` | 经典光照模型 | ✅ |
| `MeshLambertMaterial` | 漫反射光照，性能优于 Phong | ✅ |
| `MeshMatcapMaterial` | 固定光照方向，性能高 | ❌（使用贴图模拟） |
| `MeshNormalMaterial` | 法线可视化，用于调试 | ❌ |
| `ShaderMaterial` | 自定义着色器 | 取决于实现 |

```js
// 标准材质（最常用）
const material = new THREE.MeshStandardMaterial({
  color: 0x409eff,
  roughness: 0.4,           // 粗糙度（0~1）
  metalness: 0.1,           // 金属感（0~1）
  transparent: true,
  opacity: 0.8,
  side: THREE.DoubleSide,   // 双面渲染
  wireframe: false,         // 线框模式
  emissive: 0x000000,       // 自发光颜色
});
```

### 纹理

```js
const loader = new THREE.TextureLoader();
const texture = loader.load('texture.jpg');

const material = new THREE.MeshStandardMaterial({
  map: texture,                       // 颜色贴图
  normalMap: loader.load('normal.jpg'), // 法线贴图（凹凸感）
  roughnessMap: loader.load('rough.jpg'), // 粗糙度贴图
  metalnessMap: loader.load('metal.jpg'), // 金属感贴图
});
```

### 透明纹理

```js
const texture = loader.load('tree.png');
texture.premultiplyAlpha = true;      // 保留透明度
```

## 七、光源（Light）

```js
// 环境光：均匀照亮所有面，无方向
const ambient = new THREE.AmbientLight(0xffffff, 0.3);
scene.add(ambient);

// 平行光：模拟太阳光，有方向
const directional = new THREE.DirectionalLight(0xffffff, 1);
directional.position.set(5, 10, 5);
directional.castShadow = true;        // 产生阴影
scene.add(directional);

// 点光源：从一个点向四周发射（灯泡）
const point = new THREE.PointLight(0xff6600, 1, 20);
point.position.set(2, 3, 4);
scene.add(point);

// 聚光灯：锥形光束
const spot = new THREE.SpotLight(0xffffff, 1);
spot.position.set(0, 5, 0);
spot.angle = Math.PI / 6;
scene.add(spot);

// 半球光：模拟天空和地面的环境反射
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x362b25, 0.6);
scene.add(hemi);
```

**建议**：始终至少使用 `AmbientLight + DirectionalLight` 的组合，避免完全黑暗或过于平淡。

## 八、动画循环

```js
const clock = new THREE.Clock();

function animate() {
  const delta = clock.getDelta();      // 距上一帧的时间（秒）
  const elapsed = clock.getElapsedTime(); // 距开始的总时间

  // 基于时间的动画（与帧率无关）
  cube.position.y = Math.sin(elapsed) * 2;
  cube.rotation.x += delta;

  controls.update();
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

### 帧率无关的动画

始终使用 `delta` 或 `elapsed` 控制动画速度，而不是 `+= 0.01` 这种固定增量：

```js
// ❌ 60fps 时转得快，30fps 时转得慢
mesh.rotation.y += 0.01;

// ✅ 任何帧率下速度一致
mesh.rotation.y += delta * Math.PI / 2;  // 每秒转 90°
```

## 九、性能优化

### 1. 合并几何体

```js
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

// ❌ 1000 个独立网格 → 1000 次 draw call
for (let i = 0; i < 1000; i++) {
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(Math.random() * 100, 0, Math.random() * 100);
  scene.add(mesh);
}

// ✅ 合并为 1 个网格 → 1 次 draw call
const geometries = [];
for (let i = 0; i < 1000; i++) {
  const g = geo.clone();
  g.translate(Math.random() * 100, 0, Math.random() * 100);
  geometries.push(g);
}
const merged = mergeGeometries(geometries);
const mesh = new THREE.Mesh(merged, mat);
scene.add(mesh);
```

### 2. 实例化网格（InstancedMesh）

适合大量相同几何体的场景（如森林、粒子）：

```js
const count = 10000;
const instancedMesh = new THREE.InstancedMesh(geometry, material, count);
const dummy = new THREE.Object3D();

for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 200, 0, Math.random() * 200);
  dummy.rotation.set(0, Math.random() * Math.PI * 2, 0);
  dummy.scale.setScalar(0.5 + Math.random() * 1.5);
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
}
instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);
```

### 3. LOD（Level of Detail）

根据距离切换不同精度的模型：

```js
import { LOD } from 'three';

const lod = new LOD();

// 远距离（低质量）
lod.addLevel(lowPolyMesh, 50);
// 中距离
lod.addLevel(mediumPolyMesh, 20);
// 近距离（高质量）
lod.addLevel(highPolyMesh, 0);

scene.add(lod);
```

### 4. 其他优化措施

```js
// 限制帧率
renderer.setAnimationLoop(null);  // 取消内置循环
let lastTime = 0;
function animate(time) {
  if (time - lastTime >= 16) {    // ≈60fps
    lastTime = time;
    renderer.render(scene, camera);
  }
  requestAnimationFrame(animate);
}

// 降低阴影质量
directionalLight.shadow.mapSize.width = 512;
directionalLight.shadow.mapSize.height = 512;

// 关闭不需要的渲染功能
renderer.shadowMap.enabled = false;   // 不需要阴影时关闭
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 限制像素比
```

## 十、加载外部模型

Three.js 通过加载器支持多种 3D 格式：

```bash
npm install three/examples/jsm/loaders/GLTFLoader.js
```

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load('model.glb', (gltf) => {
  scene.add(gltf.scene);

  // 播放动画（如果模型包含）
  const mixer = new THREE.AnimationMixer(gltf.scene);
  gltf.animations.forEach((clip) => {
    mixer.clipAction(clip).play();
  });
});
```

## 十一、常见问题

### Q1: Three.js 包体积太大怎么办

```js
// 按需导入，不要全量引入
import { Scene, PerspectiveCamera, WebGLRenderer } from 'three';  // ✅

// 不要这样
import * as THREE from 'three';   // ❌ 全量导入，打包体积大
```

### Q2: 模型加载后看不到

```js
// 常见原因：
// 1. 模型位置在相机后面 → 调整相机位置或模型位置
// 2. 没有加光源 → 添加 AmbientLight + DirectionalLight
// 3. 模型太大/太小 → camera.position.z 可能需要调整
// 4. 忘记将模型添加到 scene → scene.add(model)
```

### Q3: 如何提高模型加载速度

```js
// 使用 DRACO 压缩
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
loader.setDRACOLoader(dracoLoader);
```

### Q4: 如何检测性能瓶颈

```js
// 打开 Chrome DevTools → Performance 录制
// 或使用 Stats.js
import Stats from 'three/addons/libs/stats.module.js';

const stats = new Stats();
document.body.appendChild(stats.dom);

function animate() {
  stats.begin();
  renderer.render(scene, camera);
  stats.end();
  requestAnimationFrame(animate);
}
```

## 十二、WebGPU 与未来

WebGPU 是 WebGL 的下一代 API，提供更底层的 GPU 控制、更低的 CPU 开销和更好的性能。

```js
// Three.js 支持 WebGPU（实验性）
import { WebGPURenderer } from 'three/addons/renderers/webgpu/WebGPURenderer.js';

const renderer = new WebGPURenderer({ antialias: true });
```

WebGPU 的优势：
- 更高效的 GPU 计算（通用计算 Compute Shader）
- 更低的 API 调用开销
- 更好的多线程支持
- 更接近现代图形 API（Vulkan/Metal/DirectX 12）

目前 Chrome、Edge、Safari 已支持，Three.js r150+ 开始实验性支持。

## 十三、推荐学习路径

1. 理解 Three.js 四大核心：Scene、Camera、Renderer、Mesh
2. 掌握内置几何体和常用材质，搭建简单场景
3. 添加光源和阴影，理解光照对材质的影响
4. 用 OrbitControls 进行交互
5. 学习纹理贴图和使用外部模型（GLTF）
6. 优化大规模场景：InstancedMesh、合并几何体、LOD
7. 了解 ShaderMaterial 和自定义着色器（进阶）
8. 关注 WebGPU 的发展
