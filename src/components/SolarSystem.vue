<template>
  <div class="solar-system-wrapper">
    <!-- 3D渲染容器 -->
    <div ref="container" class="solar-container"></div>
    
    <!-- 加载提示 -->
    <div v-if="loading" class="loading-overlay">
      <div class="loading-content">
        <div class="spinner"></div>
        <p>加载太阳系模型中... {{ loadingProgress }}%</p>
      </div>
    </div>
    
    <!-- 产品导航面板 -->
    <div class="product-panel" :class="{ collapsed: panelCollapsed }">
      <div class="panel-header" @click="panelCollapsed = !panelCollapsed">
        <h3>{{ panelCollapsed ? '☰' : '产品导航' }}</h3>
        <span class="toggle-icon">{{ panelCollapsed ? '◀' : '▶' }}</span>
      </div>
      
      <div v-if="!panelCollapsed" class="panel-content">
        <div 
          v-for="product in products" 
          :key="product.name"
          class="product-item"
          :class="{ active: selectedProduct === product.name }"
          @click="navigateToPlanet(product)"
        >
          <div class="product-icon">{{ product.icon }}</div>
          <div class="product-info">
            <div class="product-name">{{ product.displayName }}</div>
            <div class="product-desc">{{ product.product }}</div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 信息提示 -->
    <div v-if="currentInfo" class="info-tooltip">
      <h4>{{ currentInfo.displayName }}</h4>
      <p>{{ currentInfo.product }}</p>
      <button @click="currentInfo = null">关闭</button>
    </div>
    
    <!-- 控制提示 -->
    <div class="controls-hint">
      <p>🖱️ 左键拖拽旋转 | 滚轮缩放 | 右键平移</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { FBXLoader } from "three/examples/jsm/loaders/FBXLoader.js";
import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader.js";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
import { SSAOPass } from "three/examples/jsm/postprocessing/SSAOPass.js";
import gsap from "gsap";

const container = ref(null);
const loading = ref(true);
const loadingProgress = ref(0);
const panelCollapsed = ref(false);
const selectedProduct = ref(null);
const currentInfo = ref(null);

let scene, camera, renderer, controls;
let solarSystemModel = null;
let planetObjects = {}; // 存储找到的行星对象
let animationId = null;
let composer = null; // 后处理合成器
let envMap = null; // 环境贴图

// 产品与行星的映射关系
const products = [
  { name: "Sun", displayName: "太阳 ☀️", icon: "☀️", product: "核心产品 - 能源解决方案", keywords: ["sun", "sol"] },
  { name: "Mercury", displayName: "水星 ☿️", icon: "☿️", product: "产品A - 快速响应系统", keywords: ["mercury"] },
  { name: "Venus", displayName: "金星 ♀️", icon: "♀️", product: "产品B - 高温材料", keywords: ["venus"] },
  { name: "Earth", displayName: "地球 🌍", icon: "🌍", product: "产品C - 智能地球方案", keywords: ["earth", "tierra"] },
  { name: "Mars", displayName: "火星 ♂️", icon: "♂️", product: "产品D - 探索系统", keywords: ["mars"] },
  { name: "Jupiter", displayName: "木星 ♃", icon: "♃", product: "产品E - 大数据平台", keywords: ["jupiter"] },
  { name: "Saturn", displayName: "土星 ♄", icon: "♄", product: "产品F - 环形网络", keywords: ["saturn"] },
  { name: "Uranus", displayName: "天王星 ♅", icon: "♅", product: "产品G - 创新技术", keywords: ["uranus"] },
  { name: "Neptune", displayName: "海王星 ♆", icon: "♆", product: "产品H - 深度计算", keywords: ["neptune"] }
];

onMounted(() => {
  initScene();
  setupEnvironmentMap(); // 添加环境映射
  setupPostProcessing(); // 添加后处理效果
  loadSolarSystem();
  animate();
  
  window.addEventListener("resize", onWindowResize);
  window.addEventListener("click", onCanvasClick);
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", onWindowResize);
  window.removeEventListener("click", onCanvasClick);
  
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
  
  if (renderer) {
    renderer.dispose();
  }
});

function initScene() {
  // 创建场景
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x000510);
  
  // 创建相机
  const width = window.innerWidth;
  const height = window.innerHeight;
  // 🔧 优化near/far平面设置，避免深度缓冲精度问题
  // near不要太小，far不要太大，比例保持在1:1000以内最佳
  camera = new THREE.PerspectiveCamera(45, width / height, 1, 2000);
  camera.position.set(0, 30, 80);
  
  // 创建渲染器 - 高质量设置
  renderer = new THREE.WebGLRenderer({ 
    antialias: true,
    alpha: false,
    powerPreference: "high-performance", // 使用高性能GPU
    precision: "highp" // 高精度着色器
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 限制最大为2，避免性能问题
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap; // 柔和阴影
  
  // 🎨 启用物理正确的光照和色调映射，提升渲染质量
  renderer.physicallyCorrectLights = true;
  renderer.outputEncoding = THREE.sRGBEncoding; // 正确的颜色空间
  renderer.toneMapping = THREE.ACESFilmicToneMapping; // 电影级色调映射
  renderer.toneMappingExposure = 1.0;
  
  container.value.appendChild(renderer.domElement);
  
  console.log('🎨 渲染器配置:');
  console.log('  像素比:', renderer.getPixelRatio());
  console.log('  输出编码:', 'sRGB');
  console.log('  色调映射:', 'ACES Filmic');
  
  // 添加轨道控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  // 🔧 优化缩放范围，避免过近或过远导致的问题
  controls.minDistance = 5; // 最小距离（避免穿模）
  controls.maxDistance = 500; // 最大距离（合理范围）
  controls.enableZoom = true;
  controls.enableRotate = true;
  controls.enablePan = true;
  controls.zoomSpeed = 1.0; // 适中的缩放速度
  controls.rotateSpeed = 0.8; // 适中的旋转速度
  controls.screenSpacePanning = true; // 屏幕空间平移，更直观
  controls.mouseButtons = {
    LEFT: THREE.MOUSE.ROTATE,
    MIDDLE: THREE.MOUSE.DOLLY,
    RIGHT: THREE.MOUSE.PAN
  }
  
  // 💡 优化光源设置，提升视觉质量
  // 环境光 - 提供基础照明
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3);
  scene.add(ambientLight);
  
  // 主光源（太阳）- 使用点光源
  const pointLight = new THREE.PointLight(0xffffff, 2.5, 2000);
  pointLight.position.set(0, 0, 0);
  pointLight.castShadow = true;
  pointLight.shadow.mapSize.width = 2048;  // 高分辨率阴影
  pointLight.shadow.mapSize.height = 2048;
  pointLight.shadow.camera.near = 0.5;
  pointLight.shadow.camera.far = 500;
  scene.add(pointLight);
  
  // 辅助光源 - 增加细节可见度
  const hemisphereLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.6);
  scene.add(hemisphereLight);
  
  console.log('💡 光源配置:');
  console.log('  环境光强度: 0.3');
  console.log('  主光源强度: 2.5');
  console.log('  阴影分辨率: 2048x2048');
  
  // 添加星空背景
  createStarField();
}

function createStarField() {
  const starGeometry = new THREE.BufferGeometry();
  const starVertices = [];
  
  for (let i = 0; i < 3000; i++) {
    const x = (Math.random() - 0.5) * 4000;
    const y = (Math.random() - 0.5) * 4000;
    const z = (Math.random() - 0.5) * 4000;
    starVertices.push(x, y, z);
  }
  
  starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starVertices, 3));
  const starMaterial = new THREE.PointsMaterial({ 
    color: 0xffffff, 
    size: 2,
    transparent: true,
    opacity: 0.8
  });
  
  const stars = new THREE.Points(starGeometry, starMaterial);
  scene.add(stars);
}

// 🌟 设置环境映射（基于图像的光照 - IBL）
function setupEnvironmentMap() {
  console.log('🌟 设置环境映射...');
  
  // 创建程序化环境贴图（如果没有HDRI文件）
  const pmremGenerator = new THREE.PMREMGenerator(renderer);
  pmremGenerator.compileEquirectangularShader();
  
  // 创建一个简单的渐变环境贴图
  const envScene = new THREE.Scene();
  const envLight1 = new THREE.DirectionalLight(0x8899ff, 0.5);
  envLight1.position.set(1, 1, 1);
  const envLight2 = new THREE.DirectionalLight(0xff8844, 0.3);
  envLight2.position.set(-1, -1, -1);
  envScene.add(envLight1, envLight2);
  
  envMap = pmremGenerator.fromScene(envScene).texture;
  scene.environment = envMap;
  
  console.log('  ✓ 环境映射已设置');
  
  pmremGenerator.dispose();
}

// 🎨 设置后处理效果（像Sketchfab一样）
function setupPostProcessing() {
  console.log('🎨 设置后处理效果...');
  
  // 创建效果合成器
  composer = new EffectComposer(renderer);
  
  // 1. 基础渲染通道
  const renderPass = new RenderPass(scene, camera);
  composer.addPass(renderPass);
  
  // 2. SSAO（屏幕空间环境光遮蔽）- 增加深度感和细节
  const ssaoPass = new SSAOPass(scene, camera);
  ssaoPass.kernelRadius = 4; // 降低半径，避免过度
  ssaoPass.minDistance = 0.005;
  ssaoPass.maxDistance = 0.05;
  ssaoPass.output = SSAOPass.OUTPUT.Default;
  composer.addPass(ssaoPass);
  
  // 3. Bloom（泛光）- 让发光体更真实
  const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    0.2,  // 强度（降低，避免过度发光）
    0.3,  // 半径（降低）
    0.9   // 阈值（提高，只有很亮的部分才发光）
  );
  composer.addPass(bloomPass);
  
  console.log('  ✓ 后处理效果已启用:');
  console.log('    - SSAO (环境光遮蔽)');
  console.log('    - Bloom (泛光效果)');
}

// 纹理映射配置
const textureConfig = {
  'Earth': {
    albedo: '/models/high-resolution-solar-system/textures/EartAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/EarthNormal.jpeg',
    roughness: '/models/high-resolution-solar-system/textures/EarthRoughness.jpeg',
    cloud: '/models/high-resolution-solar-system/textures/cloud_combined_2048.png'
  },
  'Moon': {
    albedo: '/models/high-resolution-solar-system/textures/MoonAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/MoonNormal.jpeg'
  },
  'Sun': {
    albedo: '/models/high-resolution-solar-system/textures/SunAlbedo.jpeg',
    emissive: '/models/high-resolution-solar-system/textures/SunAlbedo.jpeg'
  },
  'Mercury': {
    albedo: '/models/high-resolution-solar-system/textures/MercuryAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/MercuryNormal.jpeg',
    emissive: '/models/high-resolution-solar-system/textures/MercuryEmissive.png'
  },
  'Venus': {
    albedo: '/models/high-resolution-solar-system/textures/VenusAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/VenusNormal.jpeg'
  },
  'Mars': {
    albedo: '/models/high-resolution-solar-system/textures/MarsAlbedoo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/MarsNormal.jpeg'
  },
  'Jupiter': {
    albedo: '/models/high-resolution-solar-system/textures/JupiterAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/JupiterNormal.jpeg',
    alpha: '/models/high-resolution-solar-system/textures/JupiterAlpha.png'
  },
  'Saturn': {
    albedo: '/models/high-resolution-solar-system/textures/SaturnAlbedo.png'
  },
  'Uranus': {
    albedo: '/models/high-resolution-solar-system/textures/UranusAlbedo.png',
    normal: '/models/high-resolution-solar-system/textures/UranusNormal.jpeg'
  },
  'Neptune': {
    albedo: '/models/high-resolution-solar-system/textures/NeptuneColor.jpeg'
  },
  'Pluto': {
    albedo: '/models/high-resolution-solar-system/textures/PlutoAlbedo.jpeg',
    normal: '/models/high-resolution-solar-system/textures/PlutoNormal.jpeg'
  }
};

// 自动调整相机位置以完整查看模型
function fitCameraToModel(model) {
  const box = new THREE.Box3().setFromObject(model);
  const size = box.getSize(new THREE.Vector3());
  const center = box.getCenter(new THREE.Vector3());
  
  const maxDim = Math.max(size.x, size.y, size.z);
  const fov = camera.fov * (Math.PI / 180);
  let cameraDistance = Math.abs(maxDim / Math.sin(fov / 2)) * 1.5; // 1.5倍距离确保完整显示
  
  // 设置相机位置
  camera.position.set(
    center.x + cameraDistance * 0.5,
    center.y + cameraDistance * 0.5,
    center.z + cameraDistance
  );
  
  // 设置控制器目标为模型中心
  controls.target.copy(center);
  controls.update();
  
  console.log('📷 相机自动调整:');
  console.log('  模型中心:', center);
  console.log('  相机距离:', cameraDistance);
  console.log('  相机位置:', camera.position);
}

// 添加调试辅助器
function addDebugHelpers(model) {
  // 添加坐标轴辅助器（红色=X轴，绿色=Y轴，蓝色=Z轴）
  const axesHelper = new THREE.AxesHelper(50);
  scene.add(axesHelper);
  
  // 添加网格辅助器
  const gridHelper = new THREE.GridHelper(100, 20, 0x444444, 0x222222);
  scene.add(gridHelper);
  
  // 添加模型包围盒辅助器
  const box = new THREE.Box3().setFromObject(model);
  const boxHelper = new THREE.Box3Helper(box, 0x00ff00);
  scene.add(boxHelper);
  
  console.log('🔧 调试辅助器已添加:');
  console.log('  ✓ 坐标轴（红=X, 绿=Y, 蓝=Z）');
  console.log('  ✓ 地面网格');
  console.log('  ✓ 包围盒（绿色）');
}

// 高质量纹理加载函数
function loadTextureWithQuality(url) {
  const textureLoader = new THREE.TextureLoader();
  const texture = textureLoader.load(
    url,
    (tex) => {
      console.log(`    ✓ 纹理加载成功: ${url.split('/').pop()}`);
    },
    undefined,
    (err) => {
      console.warn(`    ⚠ 纹理加载失败: ${url}`, err);
    }
  );
  
  // 🎨 关键！设置各向异性过滤，大幅提升清晰度
  const maxAnisotropy = renderer.capabilities.getMaxAnisotropy();
  texture.anisotropy = maxAnisotropy; // 使用硬件支持的最大值（通常是16）
  
  // 🎨 设置纹理过滤模式
  texture.minFilter = THREE.LinearMipmapLinearFilter; // 最佳质量的缩小过滤
  texture.magFilter = THREE.LinearFilter; // 放大过滤
  
  // 🎨 设置纹理包裹模式
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;
  
  // 🎨 启用mipmap生成
  texture.generateMipmaps = true;
  
  // 🎨 设置颜色空间（对于基础颜色贴图）
  texture.encoding = THREE.sRGBEncoding;
  
  return texture;
}

function applyTexturesToMesh(mesh, planetName) {
  const config = textureConfig[planetName];
  if (!config) return;
  
  console.log(`  🎨 加载${planetName}的纹理...`);
  
  const material = new THREE.MeshStandardMaterial({
    // 材质属性优化
    metalness: 0.0,  // 行星不是金属
    roughness: 0.8,  // 适中的粗糙度
    envMapIntensity: 1.5, // 🌟 环境映射强度（增加反射效果）
    flatShading: false, // 🔧 关键！启用平滑着色，消除马赛克效果
  });
  
  // 加载基础颜色贴图（使用高质量加载）
  if (config.albedo) {
    material.map = loadTextureWithQuality(config.albedo);
  }
  
  // 加载法线贴图（不使用sRGB编码）
  if (config.normal) {
    const normalMap = loadTextureWithQuality(config.normal);
    normalMap.encoding = THREE.LinearEncoding; // 法线贴图使用线性编码
    material.normalMap = normalMap;
    // 🎨 增强法线效果，让表面细节更明显
    material.normalScale = new THREE.Vector2(2.0, 2.0); // 增加法线强度
  }
  
  // 加载粗糙度贴图（不使用sRGB编码）
  if (config.roughness) {
    const roughnessMap = loadTextureWithQuality(config.roughness);
    roughnessMap.encoding = THREE.LinearEncoding;
    material.roughnessMap = roughnessMap;
  }
  
  // 加载自发光贴图（用于太阳等发光体）
  if (config.emissive) {
    material.emissiveMap = loadTextureWithQuality(config.emissive);
    material.emissive = new THREE.Color(0xffffff);
    material.emissiveIntensity = planetName === 'Sun' ? 1.5 : 0.3;
  }
  
  // 加载透明度贴图
  if (config.alpha) {
    const alphaMap = loadTextureWithQuality(config.alpha);
    alphaMap.encoding = THREE.LinearEncoding;
    material.alphaMap = alphaMap;
    material.transparent = true;
  }
  
  // 应用材质
  mesh.material = material;
  mesh.material.needsUpdate = true;
  
  console.log(`  ✓ ${planetName}纹理应用完成（各向异性过滤: ${renderer.capabilities.getMaxAnisotropy()}x）`);
}

function loadSolarSystem() {
  const loader = new FBXLoader();
  
  loader.load(
    '/models/high-resolution-solar-system/source/Earth&Moon.fbx',
    (fbx) => {
      solarSystemModel = fbx;
      
      console.log('========== FBX 模型结构调试信息 ==========');
      console.log('模型根节点:', fbx);
      console.log('原始缩放:', fbx.scale);
      console.log('原始位置:', fbx.position);
      console.log('原始旋转:', fbx.rotation);
      
      // 🔧 修复1: 计算模型的包围盒，用于自动缩放和居中
      const box = new THREE.Box3().setFromObject(fbx);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3());
      
      console.log('模型包围盒中心:', center);
      console.log('模型包围盒尺寸:', size);
      
      // 🔧 修复2: 将模型居中（移动到原点）
      fbx.position.x = -center.x;
      fbx.position.y = -center.y;
      fbx.position.z = -center.z;
      
      // 🔧 修复3: 根据模型大小自动缩放到合适的尺寸
      // 目标尺寸：让最大维度为约30个单位
      const maxDim = Math.max(size.x, size.y, size.z);
      const targetSize = 30;
      const scale = targetSize / maxDim;
      
      console.log('最大维度:', maxDim);
      console.log('应用缩放:', scale);
      
      // 统一缩放，避免扭曲
      fbx.scale.set(scale, scale, scale);
      
      // 🔧 修复4: 确保变换矩阵正确更新
      fbx.updateMatrix();
      fbx.updateMatrixWorld(true);
      
      // 遍历模型，找到各个行星
      let meshIndex = 0;
      solarSystemModel.traverse((child) => {
        if (child.isMesh) {
          meshIndex++;
          
          // 获取世界坐标
          const worldPos = new THREE.Vector3();
          child.getWorldPosition(worldPos);
          
          // 获取包围盒大小
          const box = new THREE.Box3().setFromObject(child);
          const size = box.getSize(new THREE.Vector3());
          
          // 🎯 获取几何体信息
          const geometry = child.geometry;
          const faceCount = geometry.index ? geometry.index.count / 3 : geometry.attributes.position.count / 3;
          const vertexCount = geometry.attributes.position.count;
          
          // 详细打印每个网格信息
          console.log(`\n[${meshIndex}] 对象信息:`);
          console.log('  名称:', child.name || '(未命名)');
          console.log('  类型:', child.type);
          console.log('  位置:', `x:${worldPos.x.toFixed(2)}, y:${worldPos.y.toFixed(2)}, z:${worldPos.z.toFixed(2)}`);
          console.log('  尺寸:', `${size.x.toFixed(2)} x ${size.y.toFixed(2)} x ${size.z.toFixed(2)}`);
          console.log('  面数:', Math.floor(faceCount));
          console.log('  顶点数:', vertexCount);
          console.log('  父级:', child.parent?.name || '根节点');
          
          // 🔧 关键修复：计算平滑法线，消除马赛克效果
          if (geometry.attributes.position) {
            // 重新计算顶点法线（平滑着色）
            geometry.computeVertexNormals();
            console.log('  ✓ 已重新计算平滑法线');
            
            // 检查面数是否过少
            if (faceCount < 500) {
              console.warn(`  ⚠️ 警告：面数较少(${Math.floor(faceCount)})，可能显示不够圆滑`);
              console.warn('  💡 建议：使用更高多边形的模型或在3D软件中细分');
            } else {
              console.log(`  ✓ 面数充足(${Math.floor(faceCount)})，应该显示圆滑`);
            }
          }
          
          // 尝试匹配行星并应用纹理
          let matched = false;
          products.forEach(product => {
            product.keywords.forEach(keyword => {
              const name = child.name.toLowerCase();
              if (name.includes(keyword)) {
                planetObjects[product.name] = child;
                console.log(`  ✓ 匹配到: ${product.displayName}`);
                
                // 应用对应的纹理
                applyTexturesToMesh(child, product.name);
                matched = true;
              }
            });
          });
          
          // 检查是否是月球
          if (!matched && child.name.toLowerCase().includes('moon')) {
            planetObjects['Moon'] = child;
            console.log(`  ✓ 匹配到: 月球`);
            applyTexturesToMesh(child, 'Moon');
            matched = true;
          }
          
          if (!matched) {
            console.log('  ⚠ 未匹配到任何行星');
          }
          
          // 确保材质正确接收光照
          child.castShadow = true;
          child.receiveShadow = true;
        }
        
        // 也打印组节点
        if (child.isGroup || child.isObject3D) {
          console.log(`\n[Group] ${child.name || '(未命名组)'} - 类型: ${child.type}`);
        }
      });
      
      console.log('\n========== 匹配结果汇总 ==========');
      console.log('已匹配的行星:', Object.keys(planetObjects));
      console.log('未匹配的产品:', products.filter(p => !planetObjects[p.name]).map(p => p.displayName));
      console.log('=====================================\n');
      
      scene.add(solarSystemModel);
      
      // 🔧 修复5: 模型加载完成后，自动调整相机位置以完整查看模型
      fitCameraToModel(solarSystemModel);
      
      // 🔧 可选：添加调试辅助器（开发时使用，生产环境可注释）
      addDebugHelpers(solarSystemModel);
      
      loading.value = false;
      
      console.log('✓ Solar system loaded successfully');
    },
    (progress) => {
      // 加载进度
      const percent = (progress.loaded / progress.total) * 100;
      loadingProgress.value = Math.round(percent);
    },
    (error) => {
      console.error('Error loading model:', error);
      loading.value = false;
      alert('加载太阳系模型失败，请检查文件路径是否正确');
    }
  );
}

function animate() {
  animationId = requestAnimationFrame(animate);
  
  // 让整个太阳系缓慢旋转（可选）
  if (solarSystemModel) {
    // solarSystemModel.rotation.y += 0.0001; // 取消注释以启用自动旋转
  }
  
  if (controls) controls.update();
  
  // 使用后处理合成器渲染（如果启用）
  if (composer) {
    composer.render();
  } else if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}

function navigateToPlanet(product) {
  selectedProduct.value = product.name;
  currentInfo.value = product;
  
  const planetObject = planetObjects[product.name];
  
  if (!planetObject) {
    console.error(`❌ 未找到 ${product.displayName}`);
    console.log('当前已识别的行星:', Object.keys(planetObjects));
    alert(`未在模型中找到 ${product.displayName}\n\n请按 F12 打开控制台查看模型结构信息\n在控制台中搜索对应的行星名称`);
    return;
  }
  
  console.log(`\n🎯 导航到 ${product.displayName}`);
  
  // 获取行星的世界坐标
  const worldPosition = new THREE.Vector3();
  planetObject.getWorldPosition(worldPosition);
  console.log('  行星位置:', worldPosition);
  
  // 计算包围盒以确定合适的观察距离
  const box = new THREE.Box3().setFromObject(planetObject);
  const size = box.getSize(new THREE.Vector3());
  const maxDim = Math.max(size.x, size.y, size.z);
  const distance = Math.max(maxDim * 5, 10); // 至少距离10个单位
  
  console.log('  行星尺寸:', size);
  console.log('  观察距离:', distance);
  
  // 计算目标相机位置（在行星前方偏上）
  const offset = new THREE.Vector3(
    distance * 0.3,
    distance * 0.5,
    distance
  );
  
  const targetPosition = new THREE.Vector3().copy(worldPosition).add(offset);
  
  console.log('  目标相机位置:', targetPosition);
  console.log('  当前相机位置:', camera.position);
  
  // 使用GSAP实现平滑相机移动
  gsap.to(camera.position, {
    x: targetPosition.x,
    y: targetPosition.y,
    z: targetPosition.z,
    duration: 2,
    ease: "power2.inOut",
    onStart: () => {
      console.log('  开始移动相机...');
    },
    onUpdate: () => {
      controls.target.copy(worldPosition);
      controls.update();
    },
    onComplete: () => {
      console.log('  ✓ 相机移动完成');
    }
  });
  
  // 同时更新控制器目标
  gsap.to(controls.target, {
    x: worldPosition.x,
    y: worldPosition.y,
    z: worldPosition.z,
    duration: 2,
    ease: "power2.inOut"
  });
}

function onCanvasClick(event) {
  // 检查是否点击在canvas上（不是UI元素）
  if (event.target !== renderer.domElement) return;
  
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2();
  
  const rect = renderer.domElement.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
  
  raycaster.setFromCamera(mouse, camera);
  
  // 检查与所有行星的相交
  const planetMeshes = Object.values(planetObjects);
  const intersects = raycaster.intersectObjects(planetMeshes, true);
  
  if (intersects.length > 0) {
    const clickedObject = intersects[0].object;
    
    // 找到对应的产品
    for (const [planetName, planetObj] of Object.entries(planetObjects)) {
      if (clickedObject === planetObj || clickedObject.parent === planetObj) {
        const product = products.find(p => p.name === planetName);
        if (product) {
          currentInfo.value = product;
          selectedProduct.value = product.name;
          console.log(`Clicked on ${product.displayName}`);
        }
        break;
      }
    }
  }
}

function onWindowResize() {
  if (!camera || !renderer) return;
  
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  
  // 更新后处理合成器的大小
  if (composer) {
    composer.setSize(window.innerWidth, window.innerHeight);
  }
}
</script>

<style scoped>
.solar-system-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  padding: 0;
}

.solar-container {
  width: 100%;
  height: 100%;
  background-color: #000;
  position: absolute;
  top: 0;
  left: 0;
}

/* 加载动画 */
.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 5, 16, 0.95);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.loading-content {
  text-align: center;
  color: white;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(255, 255, 255, 0.1);
  border-top: 4px solid #42b883;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 20px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* 产品导航面板 */
.product-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 320px;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
  transition: all 0.3s ease;
  z-index: 100;
  max-height: 80vh;
  overflow: hidden;
}

.product-panel.collapsed {
  width: 60px;
}

.panel-header {
  padding: 15px 20px;
  background: rgba(66, 184, 131, 0.2);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  user-select: none;
}

.product-panel.collapsed .panel-header {
  padding: 15px 10px;
  justify-content: center;
}

.panel-header h3 {
  margin: 0;
  color: #42b883;
  font-size: 18px;
  font-weight: 600;
}

.toggle-icon {
  color: #42b883;
  font-size: 14px;
}

.product-panel.collapsed .toggle-icon {
  display: none;
}

.panel-content {
  max-height: calc(80vh - 60px);
  overflow-y: auto;
  padding: 10px;
}

.panel-content::-webkit-scrollbar {
  width: 6px;
}

.panel-content::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.panel-content::-webkit-scrollbar-thumb {
  background: rgba(66, 184, 131, 0.5);
  border-radius: 3px;
}

.product-item {
  display: flex;
  align-items: center;
  padding: 12px;
  margin: 8px 0;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.product-item:hover {
  background: rgba(66, 184, 131, 0.2);
  border-color: rgba(66, 184, 131, 0.5);
  transform: translateX(-5px);
}

.product-item.active {
  background: rgba(66, 184, 131, 0.3);
  border-color: #42b883;
}

.product-icon {
  font-size: 32px;
  margin-right: 12px;
  min-width: 40px;
  text-align: center;
}

.product-info {
  flex: 1;
}

.product-name {
  color: white;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 4px;
}

.product-desc {
  color: rgba(255, 255, 255, 0.6);
  font-size: 13px;
}

/* 信息提示框 */
.info-tooltip {
  position: absolute;
  bottom: 80px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.9);
  backdrop-filter: blur(10px);
  padding: 20px 30px;
  border-radius: 12px;
  border: 1px solid rgba(66, 184, 131, 0.5);
  color: white;
  z-index: 100;
  min-width: 300px;
  text-align: center;
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.info-tooltip h4 {
  margin: 0 0 10px 0;
  color: #42b883;
  font-size: 20px;
}

.info-tooltip p {
  margin: 0 0 15px 0;
  color: rgba(255, 255, 255, 0.8);
}

.info-tooltip button {
  background: #42b883;
  color: white;
  border: none;
  padding: 8px 20px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s ease;
}

.info-tooltip button:hover {
  background: #35a372;
  transform: scale(1.05);
}

/* 控制提示 */
.controls-hint {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.7);
  padding: 10px 20px;
  border-radius: 20px;
  color: rgba(255, 255, 255, 0.6);
  font-size: 14px;
  z-index: 50;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .product-panel {
    width: 280px;
    top: 10px;
    right: 10px;
  }
  
  .info-tooltip {
    min-width: 250px;
    padding: 15px 20px;
  }
  
  .controls-hint {
    font-size: 12px;
    padding: 8px 15px;
    bottom: 10px;
  }
}
</style>
