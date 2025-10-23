# FBX 模型加载指南

## 🎯 问题原因

从 [Fab.com](https://www.fab.com/listings/f8e5507c-76d2-44f5-97b2-d22c264dc4ab) 下载的模型显示样式不对的主要原因：

### 1. **FBX 文件不自动加载纹理**
- FBX 格式与 GLB/GLTF 不同，它不会将纹理嵌入到文件中
- FBX 文件内的纹理路径引用通常是绝对路径或相对路径，与你的项目结构不匹配
- 需要手动加载和应用纹理贴图

### 2. **材质系统不同**
- FBX 可能使用 Phong/Lambert 等旧材质
- Three.js 现在推荐使用 PBR（物理渲染）材质（MeshStandardMaterial）
- 需要重新创建材质并应用纹理

## ✅ 已完成的修改

### 1. 更换加载器
```javascript
// 从 GLTFLoader 改为 FBXLoader
import { FBXLoader } from "three/examples/jsm/loaders/FBXLoader.js";
```

### 2. 添加纹理配置映射
创建了 `textureConfig` 对象，将每个行星与其纹理文件对应：
```javascript
const textureConfig = {
  'Earth': {
    albedo: '/models/high-resolution-solar-system/textures/EartAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/EarthNormal.jpeg',
    roughness: '/models/high-resolution-solar-system/textures/EarthRoughness.jpeg'
  },
  // ... 其他行星
}
```

### 3. 创建纹理应用函数
```javascript
function applyTexturesToMesh(mesh, planetName) {
  // 自动为每个行星加载并应用对应的纹理贴图
  // 支持：基础颜色、法线、粗糙度、自发光、透明度等贴图
}
```

### 4. 自动识别并应用纹理
在模型加载时，自动识别每个mesh并应用对应的纹理。

## 📋 纹理类型说明

| 纹理类型 | 作用 | 文件命名 |
|---------|------|---------|
| **Albedo** (基础色) | 物体的基本颜色 | `*Albedo.jpeg` |
| **Normal** (法线) | 增加表面细节，不增加面数 | `*Normal.jpeg` |
| **Roughness** (粗糙度) | 控制表面光滑度 | `*Roughness.jpeg` |
| **Emissive** (自发光) | 让物体发光（如太阳） | `*Emissive.png` |
| **Alpha** (透明度) | 控制透明效果 | `*Alpha.png` |

## 🔧 当前配置

### 当前加载的文件
```javascript
'/models/high-resolution-solar-system/source/Earth&Moon.fbx'
```

这个文件只包含**地球和月球**，如果你需要加载完整的太阳系，可能需要：

### 选项 1：单独的 FBX 文件
如果下载包含多个 FBX 文件（每个行星一个），你需要：
```javascript
// 分别加载每个行星
loader.load('/models/high-resolution-solar-system/source/Sun.fbx', (fbx) => {...});
loader.load('/models/high-resolution-solar-system/source/Mercury.fbx', (fbx) => {...});
// ... 等等
```

### 选项 2：完整太阳系的单个 FBX
如果有一个包含所有行星的 FBX 文件，修改路径：
```javascript
loader.load('/models/high-resolution-solar-system/source/SolarSystem.fbx', (fbx) => {...});
```

## 🎨 优化建议

### 1. 纹理加载优化
如果纹理文件较大，可以添加加载管理器：
```javascript
const loadingManager = new THREE.LoadingManager();
loadingManager.onProgress = (url, loaded, total) => {
  console.log(`加载中: ${loaded}/${total}`);
};

const textureLoader = new THREE.TextureLoader(loadingManager);
```

### 2. 纹理压缩
使用压缩纹理格式（如 KTX2）可以大幅减少文件大小：
```bash
# 安装工具
npm install -g gltf-pipeline

# 压缩纹理
gltf-transform etc1s input.png output.ktx2
```

### 3. 材质属性调整
根据需要调整材质参数：
```javascript
material.metalness = 0.0;  // 金属度（0=非金属，1=金属）
material.roughness = 0.8;  // 粗糙度（0=光滑，1=粗糙）
material.envMapIntensity = 1.0;  // 环境映射强度
```

## 🔍 调试技巧

### 1. 查看模型结构
打开浏览器控制台（F12），会看到详细的模型结构信息：
```
========== FBX 模型结构调试信息 ==========
[1] 对象信息:
  名称: Earth_Mesh
  类型: Mesh
  位置: x:0.00, y:0.00, z:0.00
  尺寸: 2.00 x 2.00 x 2.00
  ✓ 匹配到: 地球 🌍
  ✓ 已应用纹理: Earth
```

### 2. 检查纹理加载
如果纹理没有显示，检查：
- 纹理文件路径是否正确
- 浏览器控制台是否有 404 错误
- 纹理文件格式是否支持（推荐 JPEG/PNG）

### 3. 检查材质
在控制台中查看材质：
```javascript
// 在浏览器控制台执行
console.log(planetObjects.Earth.material);
```

## 📁 推荐的文件结构

```
public/
  └── models/
      └── high-resolution-solar-system/
          ├── source/
          │   ├── Earth&Moon.fbx       ← 当前加载的文件
          │   ├── Sun.fbx              ← 其他行星FBX（如果有）
          │   ├── Mars.fbx
          │   └── ...
          └── textures/
              ├── EartAlbedo.jpeg       ✓ 地球基础色
              ├── EarthNormal.jpeg      ✓ 地球法线
              ├── EarthRoughness.jpeg   ✓ 地球粗糙度
              ├── MoonAlbedo.jpeg       ✓ 月球基础色
              └── ...                   ✓ 其他纹理
```

## 🚀 下一步操作

### 1. **测试当前配置**
```bash
npm run dev
```
打开浏览器查看地球和月球的显示效果。

### 2. **如果有其他行星的 FBX**
检查 `source` 文件夹是否有其他 FBX 文件：
```bash
ls public/models/high-resolution-solar-system/source/
```

### 3. **加载完整太阳系**
如果有完整的太阳系 FBX 文件，告诉我文件名，我可以帮你修改加载路径。

### 4. **验证纹理**
在控制台查看是否所有纹理都正确加载：
- ✓ 表示纹理已应用
- ⚠ 表示未找到对应纹理

## ❓ 常见问题

### Q: 模型还是没有颜色？
**A:** 检查以下几点：
1. 纹理路径是否正确（区分大小写）
2. 浏览器控制台是否有纹理加载错误
3. Mesh 名称是否与 `keywords` 匹配

### Q: 模型显示全黑？
**A:** 可能是光照问题：
1. 确保场景中有光源
2. 检查材质是否正确接收光照
3. 尝试增加环境光强度

### Q: 想使用 GLB 而不是 FBX？
**A:** GLB 格式更推荐！可以使用 Blender 转换：
1. 在 Blender 中打开 FBX 文件
2. File → Export → glTF 2.0 (.glb)
3. 选择 "GLB Binary" 格式
4. 勾选 "Apply Modifiers" 和 "Include Textures"

### Q: 如何优化性能？
**A:** 
1. 压缩纹理（使用 2K 而不是 4K）
2. 使用 LOD（细节层次）系统
3. 启用视锥体剔除
4. 合并相同材质的网格

## 📞 需要帮助？

如果遇到问题，请提供：
1. 浏览器控制台的完整输出
2. `source` 文件夹中的所有 FBX 文件列表
3. 模型在原网站上的预期效果截图

---

💡 **提示**: 当前代码已经配置好了所有行星的纹理映射，一旦找到对应的 mesh 名称，就会自动应用正确的纹理！

