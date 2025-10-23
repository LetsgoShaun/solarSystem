# GLB 太阳系模型查看器使用说明

## 功能特点

### 1. 自动识别和排列
- 自动从 GLB 文件中识别所有行星（基于材质名称）
- 将行星依次水平排列展示
- 每个行星按相对大小进行缩放

### 2. 行星列表
模型包含以下天体（按太阳系顺序）：
- ☀️ 太阳 (Sun) - 材质名: material
- ☿️ 水星 (Mercury) - 材质名: Mercury
- ♀️ 金星 (Venus) - 材质名: venus  
- 🌍 地球 (Earth) - 材质名: Earth
- 🌙 月球 (Moon) - 材质名: Moon
- ♂️ 火星 (Mars) - 材质名: Mars
- ♃ 木星 (Jupiter) - 材质名: Jupiter
- ♄ 土星 (Saturn) - 材质名: Saturn
- ♅ 天王星 (Uranus) - 材质名: Uranus
- ♆ 海王星 (Neptune) - 材质名: Neptune
- ♇ 冥王星 (Pluto) - 材质名: Pluto

### 3. 交互功能
- **左键拖拽**: 旋转视角
- **滚轮**: 缩放
- **右键拖拽**: 平移
- **点击行星列表**: 聚焦到特定行星
- **自动旋转**: 所有行星会自动缓慢自转

### 4. 信息面板
右上角面板显示：
- 行星列表（可点击聚焦）
- 每个行星的加载状态
- 模型基本信息（网格数、材质数、纹理数）

## 技术实现

### 关键代码逻辑

1. **行星识别**
   ```javascript
   // 根据材质名称匹配行星
   gltf.scene.traverse((child) => {
     if (child.isMesh || child.isSkinnedMesh) {
       const materialName = child.material.name;
       const planet = planets.find(p => p.materialName === materialName);
       if (planet) {
         planetObjects[planet.name] = { mesh: child, config: planet };
       }
     }
   });
   ```

2. **行星排列**
   ```javascript
   // 将行星依次排开，间距15个单位
   const spacing = 15;
   planets.forEach((planetConfig, index) => {
     const clonedMesh = originalMesh.clone();
     clonedMesh.scale.set(scale, scale, scale);
     clonedMesh.position.set(currentX, 0, 0);
     scene.add(clonedMesh);
     currentX += spacing;
   });
   ```

3. **相机自动调整**
   ```javascript
   // 相机定位到所有行星的中心
   const centerX = totalWidth / 2;
   camera.position.set(centerX, 20, 50);
   controls.target.set(centerX, 0, 0);
   ```

## 模型信息

根据控制台输出的完整信息：
- **网格总数**: 14个
- **材质总数**: 14个  
- **纹理总数**: 25个
- **总顶点数**: 9,630
- **总面数**: 17,132

### 纹理详情
- 大部分纹理尺寸: 4096×2048
- 包含: map（基础贴图）、normalMap（法线贴图）、roughnessMap（粗糙度）、emissiveMap（发光贴图）等
- 所有纹理都嵌入在 GLB 文件中

## 性能优化

1. 原始模型设置为不可见，只显示克隆后的行星
2. 使用高性能渲染器配置
3. 自动计算合适的相机距离
4. 平滑的相机动画（使用 GSAP）

## 开发提示

如果需要调整行星的展示：
- 修改 `planets` 数组中的 `scale` 值来调整大小
- 修改 `arrangePlanets()` 中的 `spacing` 来调整间距
- 修改 `animate()` 中的旋转速度（当前为 0.005）

## 控制台输出

页面加载时会在浏览器控制台输出详细的模型信息，包括：
- 📦 GLTF 根对象信息
- 🌳 完整的场景层级结构  
- 🔷 每个网格的详细数据
- 🎨 所有材质属性
- 🖼️ 所有纹理信息
- 🌍 行星识别和匹配结果
- 🎨 行星排列信息

