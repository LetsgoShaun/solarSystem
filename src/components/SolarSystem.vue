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
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
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
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 10000);
  camera.position.set(0, 50, 150);
  
  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.shadowMap.enabled = true;
  container.value.appendChild(renderer.domElement);
  
  // 添加轨道控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 1; // 最小距离（可以很近）
  controls.maxDistance = 5000; // 最大距离（可以很远）
  controls.enableZoom = true; // 确保缩放启用
  controls.enableRotate = true; // 确保旋转启用
  controls.enablePan = true; // 确保平移启用
  controls.zoomSpeed = 2.0; // 增加缩放速度
  controls.rotateSpeed = 1.0; // 旋转速度
  controls.mouseButtons = {
    LEFT: THREE.MOUSE.ROTATE,
    MIDDLE: THREE.MOUSE.DOLLY,
    RIGHT: THREE.MOUSE.PAN
  }
  
  // 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);
  
  const pointLight = new THREE.PointLight(0xffffff, 2, 1000);
  pointLight.position.set(0, 0, 0);
  pointLight.castShadow = true;
  scene.add(pointLight);
  
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

function loadSolarSystem() {
  const loader = new GLTFLoader();
  
  loader.load(
    '/models/high_resolution_solar_system.glb',
    (gltf) => {
      solarSystemModel = gltf.scene;
      
      console.log('========== GLB 模型结构调试信息 ==========');
      console.log('模型根节点:', gltf.scene);
      
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
          
          // 详细打印每个网格信息
          console.log(`\n[${meshIndex}] 对象信息:`);
          console.log('  名称:', child.name || '(未命名)');
          console.log('  类型:', child.type);
          console.log('  位置:', `x:${worldPos.x.toFixed(2)}, y:${worldPos.y.toFixed(2)}, z:${worldPos.z.toFixed(2)}`);
          console.log('  尺寸:', `${size.x.toFixed(2)} x ${size.y.toFixed(2)} x ${size.z.toFixed(2)}`);
          console.log('  父级:', child.parent?.name || '根节点');
          
          // 尝试匹配行星
          let matched = false;
          products.forEach(product => {
            product.keywords.forEach(keyword => {
              const name = child.name.toLowerCase();
              if (name.includes(keyword)) {
                planetObjects[product.name] = child;
                console.log(`  ✓ 匹配到: ${product.displayName}`);
                matched = true;
              }
            });
          });
          
          if (!matched) {
            console.log('  ⚠ 未匹配到任何行星');
          }
          
          // 确保材质正确接收光照
          if (child.material) {
            child.material.needsUpdate = true;
          }
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
  if (renderer && scene && camera) renderer.render(scene, camera);
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
