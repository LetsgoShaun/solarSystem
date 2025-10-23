# 🪐 太阳系3D模型系统使用指南

## ✅ 已完成的功能

### 1. **完整太阳系GLB模型加载**
- ✅ 自动加载 `public/models/high_resolution_solar_system.glb`
- ✅ 显示加载进度
- ✅ 支持高分辨率模型渲染

### 2. **产品导航面板**
- ✅ 右侧浮动面板显示9个产品（太阳系9大天体）
- ✅ 可折叠/展开面板
- ✅ 每个产品显示图标、名称、描述
- ✅ 点击产品自动导航到对应行星

### 3. **相机导航动画**
- ✅ 点击产品后相机平滑移动到行星
- ✅ 自动计算合适的观察距离
- ✅ 2秒平滑过渡动画
- ✅ 相机自动对焦到行星中心

### 4. **交互功能**
- ✅ 鼠标左键拖拽：旋转视角
- ✅ 鼠标滚轮：缩放
- ✅ 鼠标右键拖拽：平移
- ✅ 直接点击3D模型中的行星：显示信息

### 5. **美观UI**
- ✅ 深色半透明面板，毛玻璃效果
- ✅ 星空背景
- ✅ 加载动画
- ✅ 响应式设计（支持手机和平板）
- ✅ 平滑动画效果

---

## 🎯 如何使用

### 启动项目
```bash
npm run dev
```

### 导航到行星
**方法1：点击产品面板**
1. 右侧会显示产品导航面板
2. 点击任意产品（如"地球 🌍"）
3. 相机会自动飞向对应的行星

**方法2：直接点击行星**
1. 用鼠标点击3D场景中的任意行星
2. 底部会弹出信息提示框
3. 显示行星名称和对应产品信息

### 控制视角
- **旋转**：鼠标左键拖拽
- **缩放**：鼠标滚轮
- **平移**：鼠标右键拖拽
- **重置视角**：刷新页面

---

## 🔧 模型结构要求

你的GLB模型需要满足以下结构（当前会自动识别）：

### 行星命名规则
模型中的网格对象（Mesh）名称应包含以下关键词之一：

| 行星 | 关键词 |
|------|--------|
| 太阳 | `sun`, `sol` |
| 水星 | `mercury` |
| 金星 | `venus` |
| 地球 | `earth`, `tierra` |
| 火星 | `mars` |
| 木星 | `jupiter` |
| 土星 | `saturn` |
| 天王星 | `uranus` |
| 海王星 | `neptune` |

**示例**：
- ✅ `Earth_Mesh` - 会被识别为地球
- ✅ `Mercury_01` - 会被识别为水星
- ✅ `SUN_SPHERE` - 会被识别为太阳
- ❌ `Planet_01` - 不会被识别（无关键词）

### 检查模型结构

打开浏览器控制台（F12），会看到：
```
Found mesh: Sun_Sphere
Found mesh: Earth_001
✓ Matched 太阳 ☀️ -> Sun_Sphere
✓ Matched 地球 🌍 -> Earth_001
...
✓ Solar system loaded successfully
Found planets: ['Sun', 'Earth', 'Mars', ...]
```

如果某个行星没有被识别到，说明模型中的命名不符合规则。

---

## 🎨 自定义配置

### 修改产品信息

编辑 `src/components/SolarSystem.vue` 中的 `products` 数组：

```javascript
const products = [
  { 
    name: "Earth",              // 内部标识（不要改）
    displayName: "地球 🌍",     // 显示名称（可改）
    icon: "🌍",                 // 图标（可改）
    product: "产品C - 你的描述", // 产品描述（可改）
    keywords: ["earth", "tierra"] // 匹配关键词（根据模型调整）
  },
  // ... 其他行星
];
```

### 调整相机距离

在 `navigateToPlanet` 函数中修改：

```javascript
const distance = maxDim * 3; // 改为 2 或 4 调整距离
```

### 修改动画速度

```javascript
gsap.to(camera.position, {
  // ...
  duration: 2,  // 改为 1（更快）或 3（更慢）
  // ...
});
```

### 修改面板位置

在 CSS 中修改 `.product-panel` 样式：

```css
.product-panel {
  top: 20px;    /* 距离顶部 */
  right: 20px;  /* 改为 left: 20px 可以放到左侧 */
  /* ... */
}
```

---

## 🐛 故障排除

### 问题1：某些行星无法定位

**原因**：模型中的对象名称不包含关键词

**解决方案1**：检查控制台日志，找到实际的对象名称
```
Found mesh: Planet_001  // 这是实际名称
```

然后修改 `keywords` 数组：
```javascript
{ 
  name: "Earth", 
  keywords: ["earth", "planet_001"] // 添加实际名称
}
```

**解决方案2**：使用Blender等3D软件重命名模型对象

### 问题2：相机距离不合适

**症状**：
- 太近：只看到行星的一部分
- 太远：行星太小看不清

**解决方案**：调整距离倍数
```javascript
const distance = maxDim * 5; // 增大倍数
```

### 问题3：点击行星没反应

**检查清单**：
- [ ] 控制台是否显示 `Matched xxx` 信息？
- [ ] 是否点击在Canvas上（不是UI面板）？
- [ ] 模型是否加载完成（loading消失）？

### 问题4：模型不显示

**检查**：
- [ ] 文件路径：`public/models/high_resolution_solar_system.glb`
- [ ] 文件名大小写是否正确
- [ ] 控制台是否有错误信息
- [ ] 模型文件是否损坏

### 问题5：面板显示不全

**解决方案**：调整面板高度
```css
.product-panel {
  max-height: 90vh; /* 增大高度 */
}
```

---

## 📊 性能优化建议

### 模型文件大小
- 建议 < 50MB
- 如果太大，使用 gltf-pipeline 压缩：
  ```bash
  npm install -g gltf-pipeline
  gltf-pipeline -i input.glb -o output.glb -d
  ```

### 渲染性能
当前配置已优化：
- ✅ 使用抗锯齿渲染
- ✅ 启用阴影（可选关闭以提升性能）
- ✅ 自适应像素比
- ✅ 视锥体裁剪

如需进一步提升性能，可以：
```javascript
renderer.shadowMap.enabled = false; // 关闭阴影
controls.enableDamping = false;     // 关闭阻尼
```

---

## 🎬 高级功能扩展

### 添加行星轨道线

```javascript
function createOrbit(radius) {
  const curve = new THREE.EllipseCurve(
    0, 0,           // 中心点
    radius, radius, // x半径, y半径
    0, 2 * Math.PI  // 起始角, 结束角
  );
  
  const points = curve.getPoints(64);
  const geometry = new THREE.BufferGeometry().setFromPoints(points);
  const material = new THREE.LineBasicMaterial({ 
    color: 0xffffff, 
    transparent: true, 
    opacity: 0.3 
  });
  
  const orbit = new THREE.Line(geometry, material);
  orbit.rotation.x = Math.PI / 2; // 旋转到水平面
  scene.add(orbit);
}
```

### 添加行星标签

```javascript
// 使用 CSS2DRenderer 添加HTML标签
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';

const labelDiv = document.createElement('div');
labelDiv.className = 'planet-label';
labelDiv.textContent = '地球';
const label = new CSS2DObject(labelDiv);
label.position.set(0, 2, 0);
planetObject.add(label);
```

### 添加太阳光晕效果

```javascript
const glowGeometry = new THREE.SphereGeometry(3.5, 32, 32);
const glowMaterial = new THREE.MeshBasicMaterial({
  color: 0xffaa00,
  transparent: true,
  opacity: 0.3,
  side: THREE.BackSide
});
const glow = new THREE.Mesh(glowGeometry, glowMaterial);
scene.add(glow);
```

---

## 📱 移动端支持

当前已支持移动端：
- ✅ 触摸拖拽旋转
- ✅ 双指缩放
- ✅ 响应式UI布局

移动端控制：
- **旋转**：单指拖拽
- **缩放**：双指捏合
- **平移**：双指拖拽

---

## ✨ 完整功能清单

- [x] 加载完整太阳系GLB模型
- [x] 产品导航面板
- [x] 点击产品导航到行星
- [x] 相机平滑动画
- [x] 直接点击行星显示信息
- [x] 星空背景
- [x] 加载进度显示
- [x] 响应式设计
- [x] 控制提示
- [x] 可折叠面板
- [x] 美观UI设计

---

## 🚀 开始体验

刷新页面，享受你的3D太阳系导航系统吧！

如有问题，请查看浏览器控制台（F12）的日志信息。

