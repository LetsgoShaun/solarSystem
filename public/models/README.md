# 3D 模型使用说明

## 📦 推荐格式：GLB

将下载的 GLB 格式模型文件放在这个目录下。

## 📋 需要的文件列表

将你下载的行星模型按照以下名称重命名后放入此目录：

- [ ] `mercury.glb` - 水星
- [ ] `venus.glb` - 金星
- [ ] `earth.glb` - 地球
- [ ] `mars.glb` - 火星
- [ ] `jupiter.glb` - 木星
- [ ] `saturn.glb` - 土星
- [ ] `uranus.glb` - 天王星
- [ ] `neptune.glb` - 海王星
- [ ] `sun.glb` - 太阳（可选）

## 🔄 回退机制

如果 GLB 模型文件不存在或加载失败，系统会自动回退到：
1. 首先尝试加载纹理贴图球体（`/textures/行星名.jpg`）
2. 如果纹理也失败，则使用纯色球体

这意味着：
- ✅ 放入 GLB 文件后会自动使用 3D 模型
- ✅ 不放 GLB 文件也能正常运行（使用球体）
- ✅ 可以混合使用（部分用模型，部分用球体）

## 📐 模型要求

### 尺寸
模型会自动缩放到合适的大小，但建议：
- 模型中心点在原点 (0, 0, 0)
- 模型朝向为正面朝向 +Z 轴
- 建议单位尺寸：1 个单位 = 行星直径

### 优化建议
- 面数：建议 1K - 10K 面
- 纹理：2K 分辨率足够
- 压缩：使用 gltf-pipeline 压缩以减小文件体积

### 文件大小
- 单个文件建议 < 5MB
- 如果文件太大，可以使用 [gltf-pipeline](https://github.com/CesiumGS/gltf-pipeline) 压缩：
  ```bash
  gltf-pipeline -i input.glb -o output.glb -d
  ```

## 🎨 材质和光照

模型会自动：
- 接收场景中的点光源（太阳）照明
- 接收环境光
- 启用阴影投射和接收

## 🔍 调试

打开浏览器控制台（F12），会看到加载日志：
- `✓ Loaded 3D model: Earth` - 成功加载模型
- `✓ Loaded texture sphere: Earth` - 回退到纹理球体
- 警告信息会显示加载失败原因

## 📁 目录结构

```
public/
  ├── models/          # 3D 模型文件（GLB格式）
  │   ├── mercury.glb
  │   ├── venus.glb
  │   ├── earth.glb
  │   └── ...
  └── textures/        # 纹理图片（回退用）
      ├── mercury.jpg
      ├── venus.jpg
      └── ...
```

## 🌐 模型资源推荐

### 免费资源
- [Sketchfab](https://sketchfab.com/) - 搜索 "planet" 或 "solar system"
- [NASA 3D Resources](https://nasa3d.arc.nasa.gov/) - NASA官方3D模型
- [Poly Haven](https://polyhaven.com/) - 高质量免费资源
- [TurboSquid Free](https://www.turbosquid.com/Search/3D-Models/free/planet)

### 格式转换
如果下载的是其他格式：
- 在线转换：https://products.aspose.app/3d/conversion
- Blender：导入模型后导出为 GLB

## ⚙️ 高级配置

如果需要调整模型的大小、位置等，可以修改 `src/components/SolarSystem.vue` 中的 `planetData` 数组：

```javascript
{ 
  name: "Earth", 
  size: 1,           // 缩放倍数
  distance: 11,      // 距离太阳的距离
  texture: "earth.jpg", 
  model: "earth.glb",   // 模型文件名
  product: "产品C" 
}
```

