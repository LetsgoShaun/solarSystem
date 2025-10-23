# 🪐 如何使用 GLB 3D 模型替换球体

## 快速开始

### 1️⃣ 下载 GLB 格式模型
✅ **推荐下载：GLB 格式**

理由：
- Three.js 官方推荐
- 文件体积最小
- 加载速度最快
- 一个文件包含所有内容（模型+纹理+动画）

### 2️⃣ 重命名文件

将下载的模型文件重命名为：

| 行星 | 文件名 |
|------|--------|
| 水星 | `mercury.glb` |
| 金星 | `venus.glb` |
| 地球 | `earth.glb` |
| 火星 | `mars.glb` |
| 木星 | `jupiter.glb` |
| 土星 | `saturn.glb` |
| 天王星 | `uranus.glb` |
| 海王星 | `neptune.glb` |

### 3️⃣ 放入目录

将重命名后的文件放入：
```
E:\sunpureCode\pre-research\solar-system\public\models\
```

### 4️⃣ 刷新浏览器

就这么简单！代码会自动：
1. 优先尝试加载 GLB 模型
2. 如果失败，回退到纹理球体
3. 在控制台显示加载状态

---

## 🎯 格式对比

| 格式 | 推荐 | 文件大小 | 加载速度 | Three.js支持 | 说明 |
|------|------|----------|----------|--------------|------|
| **GLB** | ⭐⭐⭐⭐⭐ | 小 | 快 | 优秀 | **强烈推荐！** 二进制GLTF |
| **GLTF** | ⭐⭐⭐⭐ | 中 | 中 | 优秀 | JSON格式，可能需要多个文件 |
| **FBX** | ⭐⭐ | 大 | 慢 | 需要额外loader | Autodesk格式，常用于游戏 |
| **USDZ** | ⭐ | 中 | 慢 | 支持有限 | Apple格式，主要用于AR |

---

## 🔍 验证是否成功

### 方法1：看控制台
按 `F12` 打开浏览器控制台，应该看到：
```
✓ Loaded 3D model: Earth
✓ Loaded 3D model: Mars
...
```

### 方法2：看视觉效果
- ✅ 模型有细节（不是完美的球体）
- ✅ 模型有纹理和材质
- ✅ 土星有光环
- ✅ 模型会旋转和公转

---

## 🛠️ 故障排除

### 问题1：模型太大或太小
**解决方案**：修改 `src/components/SolarSystem.vue` 中的 `size` 参数

```javascript
{ name: "Earth", size: 2, ... }  // 增大到 2 倍
```

### 问题2：模型朝向不对
**解决方案**：在代码中添加旋转

在第 95 行添加：
```javascript
model.rotation.y = Math.PI / 2; // 旋转90度
```

### 问题3：模型不显示
**检查清单**：
- [ ] 文件名是否正确（区分大小写）
- [ ] 文件是否在 `public/models/` 目录
- [ ] 浏览器控制台是否有错误信息
- [ ] 文件是否损坏（用在线查看器测试）

### 问题4：模型太暗
**解决方案**：增加环境光亮度

修改第 69 行：
```javascript
const ambientLight = new THREE.AmbientLight(0xffffff, 0.8); // 从 0.3 改为 0.8
```

---

## 📦 推荐资源网站

### 1. **Sketchfab**（最推荐）
- 网址：https://sketchfab.com/
- 搜索：`solar system planets`
- 优点：海量模型，质量高，直接下载GLB
- 许可：注意选择 CC 协议的免费模型

### 2. **NASA 3D Resources**
- 网址：https://nasa3d.arc.nasa.gov/
- 优点：官方权威，科学准确
- 缺点：可能需要转换格式

### 3. **CGTrader**
- 网址：https://www.cgtrader.com/
- 搜索：`planet 3d model`
- 优点：专业级模型
- 注意：部分需要付费

### 4. **Poly Haven**
- 网址：https://polyhaven.com/
- 优点：完全免费，高质量
- 缺点：行星模型较少

---

## 🎨 格式转换（如果需要）

如果你下载的是 **GLTF**（不是GLB）：
- GLTF 也可以直接使用，将文件扩展名改为 `.glb`
- 或者使用在线转换：https://products.aspose.app/3d/conversion/gltf-to-glb

如果你下载的是 **FBX** 或其他格式：
1. 使用 Blender（免费）：
   - 导入 FBX 文件
   - 文件 → 导出 → glTF 2.0 (.glb)
2. 或使用在线转换：https://products.aspose.app/3d/conversion

---

## 💡 优化建议

### 压缩 GLB 文件（如果文件太大）

安装工具：
```bash
npm install -g gltf-pipeline
```

压缩文件：
```bash
gltf-pipeline -i earth.glb -o earth_compressed.glb -d
```

参数说明：
- `-d`: Draco 压缩（可减小 80% 体积）
- `-i`: 输入文件
- `-o`: 输出文件

---

## 🎬 最佳实践

### 文件大小建议
- 水星、金星、火星：< 2MB
- 地球、天王星、海王星：< 3MB  
- 木星、土星：< 5MB（因为有更多细节）

### 模型质量
- 面数：2K - 10K 三角形
- 纹理：2K (2048x1024) 就很好
- 不需要 4K 纹理（会很慢）

### 命名规范
- 使用小写
- 不要有空格
- 使用 `.glb` 扩展名（不是 `.gltf`）

---

## ✅ 测试清单

下载并放置好 GLB 文件后：

- [ ] 刷新浏览器页面
- [ ] 打开控制台（F12）检查加载日志
- [ ] 确认模型正确显示
- [ ] 测试拖拽旋转视角
- [ ] 测试点击模型弹出产品信息
- [ ] 测试窗口缩放响应

---

## 📞 需要帮助？

如果遇到问题：
1. 检查浏览器控制台的错误信息
2. 确认文件路径和文件名
3. 测试文件是否损坏（用在线查看器）
4. 尝试使用不同的模型文件

---

**祝你成功！享受 3D 太阳系的视觉盛宴！** 🚀✨

