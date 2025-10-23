<template>
  <div class="glb-viewer-wrapper">
    <!-- 3D渲染容器 -->
    <div ref="container" class="viewer-container"></div>
    
    <!-- 加载提示 -->
    <div v-if="loading" class="loading-overlay">
      <div class="loading-content">
        <div class="spinner"></div>
        <p>加载 GLB 模型中... {{ loadingProgress }}%</p>
      </div>
    </div>
    
    <!-- 模型信息面板 -->
    <div class="info-panel" :class="{ collapsed: panelCollapsed }">
      <div class="panel-header" @click="panelCollapsed = !panelCollapsed">
        <h3>{{ panelCollapsed ? '📊' : '模型信息' }}</h3>
        <span class="toggle-icon">{{ panelCollapsed ? '◀' : '▶' }}</span>
      </div>
      
      <div v-if="!panelCollapsed" class="panel-content">
        <div class="info-section">
          <h4>行星列表</h4>
          <div class="planet-list">
            <div 
              v-for="(planet, index) in displayedPlanets" 
              :key="index"
              class="planet-item"
              :class="{ active: selectedPlanet === planet.name }"
              @click="focusOnPlanet(planet)"
            >
              <div class="planet-icon">{{ planet.icon }}</div>
              <div class="planet-info">
                <div class="planet-name">{{ planet.displayName }}</div>
                <div class="planet-status" v-if="planet.object">✓ 已加载</div>
                <div class="planet-status missing" v-else>✗ 未找到</div>
              </div>
            </div>
          </div>
        </div>
        
        <div class="info-section">
          <h4>基本信息</h4>
          <div class="info-item">
            <span class="label">网格数量:</span>
            <span class="value">{{ modelInfo.meshCount }}</span>
          </div>
          <div class="info-item">
            <span class="label">材质数量:</span>
            <span class="value">{{ modelInfo.materialCount }}</span>
          </div>
          <div class="info-item">
            <span class="label">纹理数量:</span>
            <span class="value">{{ modelInfo.textureCount }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 控制提示 -->
    <div class="controls-hint">
      <p>🖱️ 左键拖拽旋转 | 滚轮缩放 | 右键平移</p>
      <button v-if="isFollowingPlanet" @click="stopFollowing" class="stop-follow-btn">
        返回全景视图
      </button>
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
const selectedPlanet = ref(null);
const displayedPlanets = ref([]);
const isFollowingPlanet = ref(false); // 是否正在跟随行星

// 模型信息
const modelInfo = ref({
  meshCount: 0,
  materialCount: 0,
  textureCount: 0,
  totalVertices: 0,
  totalFaces: 0,
  meshes: []
});

let scene, camera, renderer, controls;
let loadedModel = null;
let animationId = null;
let planetObjects = {}; // 存储行星对象
let planetMeshes = []; // 存储重新排列的行星网格

// 行星配置 - 按太阳系真实顺序，调整大小便于观察所有细节
const planets = [
  { 
    name: "Sun", 
    displayName: "太阳 ☀️", 
    icon: "☀️", 
    materialName: "material", 
    scale: 0.004, // 太阳缩小
    orbitRadius: 0, // 太阳在中心
    orbitSpeed: 0,  // 不公转
    rotationSpeed: 0.001
  },
  { 
    name: "Mercury", 
    displayName: "水星 ☿️", 
    icon: "☿️", 
    materialName: "Mercury", 
    scale: 0.006, // 增大以便观察细节
    orbitRadius: 15, // 轨道半径
    orbitSpeed: 0.04, // 公转速度（距离太阳越近越快）
    rotationSpeed: 0.005
  },
  { 
    name: "Venus", 
    displayName: "金星 ♀️", 
    icon: "♀️", 
    materialName: "venus", 
    scale: 0.006,
    orbitRadius: 25,
    orbitSpeed: 0.025,
    rotationSpeed: 0.004
  },
  { 
    name: "Earth", 
    displayName: "地球 🌍", 
    icon: "🌍", 
    materialName: "Earth", 
    scale: 0.006,
    orbitRadius: 35,
    orbitSpeed: 0.02,
    rotationSpeed: 0.01
  },
  { 
    name: "Moon", 
    displayName: "月球 🌙", 
    icon: "🌙", 
    materialName: "Moon", 
    scale: 0.005, // 增大月球
    orbitRadius: 40, // 月球轨道稍远于地球
    orbitSpeed: 0.018,
    rotationSpeed: 0.008
  },
  { 
    name: "Mars", 
    displayName: "火星 ♂️", 
    icon: "♂️", 
    materialName: "Mars", 
    scale: 0.006,
    orbitRadius: 50,
    orbitSpeed: 0.015,
    rotationSpeed: 0.009
  },
  { 
    name: "Jupiter", 
    displayName: "木星 ♃", 
    icon: "♃", 
    materialName: "Jupiter", 
    scale: 0.006, // 保持较大
    orbitRadius: 70,
    orbitSpeed: 0.008,
    rotationSpeed: 0.015
  },
  { 
    name: "Saturn", 
    displayName: "土星 ♄", 
    icon: "♄", 
    materialName: "Saturn", 
    scale: 0.006, // 保持较大
    orbitRadius: 90,
    orbitSpeed: 0.005,
    rotationSpeed: 0.012
  },
  { 
    name: "Uranus", 
    displayName: "天王星 ♅", 
    icon: "♅", 
    materialName: "Uranus", 
    scale: 0.006,
    orbitRadius: 110,
    orbitSpeed: 0.003,
    rotationSpeed: 0.008
  },
  { 
    name: "Neptune", 
    displayName: "海王星 ♆", 
    icon: "♆", 
    materialName: "Neptune", 
    scale: 0.006,
    orbitRadius: 130,
    orbitSpeed: 0.002,
    rotationSpeed: 0.007
  },
  { 
    name: "Pluto", 
    displayName: "冥王星 ♇", 
    icon: "♇", 
    materialName: "Pluto", 
    scale: 0.005, // 增大冥王星
    orbitRadius: 150,
    orbitSpeed: 0.001,
    rotationSpeed: 0.003
  }
];

onMounted(() => {
  initScene();
  loadGLBModel();
  animate();
  
  window.addEventListener("resize", onWindowResize);
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", onWindowResize);
  
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
  scene.background = new THREE.Color(0x1a1a2e);
  
  // 创建相机
  const width = window.innerWidth;
  const height = window.innerHeight;
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 5000);
  camera.position.set(50, 50, 100);
  
  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ 
    antialias: true,
    alpha: false,
    powerPreference: "high-performance"
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.outputEncoding = THREE.sRGBEncoding;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;
  
  container.value.appendChild(renderer.domElement);
  
  // 添加轨道控制器
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 1;
  controls.maxDistance = 1000;
  controls.screenSpacePanning = true;
  
  // 当用户手动操作相机时，停止跟随行星
  controls.addEventListener('start', () => {
    if (isFollowingPlanet.value) {
      isFollowingPlanet.value = false;
      console.log('⚠️ 用户手动操作，停止跟随行星');
    }
  });
  
  // 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);
  
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1.0);
  directionalLight.position.set(10, 10, 10);
  directionalLight.castShadow = true;
  directionalLight.shadow.mapSize.width = 2048;
  directionalLight.shadow.mapSize.height = 2048;
  scene.add(directionalLight);
  
  const hemisphereLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.6);
  scene.add(hemisphereLight);
  
  console.log('✓ 场景初始化完成');
}

function loadGLBModel() {
  const loader = new GLTFLoader();
  
  console.log('\n========================================');
  console.log('🔍 开始加载 GLB 模型');
  console.log('========================================\n');
  
  loader.load(
    '/models/high_resolution_solar_system.glb',
    (gltf) => {
      console.log('\n========================================');
      console.log('✅ GLB 模型加载成功！');
      console.log('========================================\n');
      
      loadedModel = gltf.scene;
      
      // 隐藏原始模型（我们会重新排列）
      loadedModel.visible = false;
      
      // ===== 打印 GLTF 根信息 =====
      console.log('📦 GLTF 根对象信息:');
      console.log('  - GLTF 对象:', gltf);
      console.log('  - Scene:', gltf.scene);
      console.log('  - Scenes 数量:', gltf.scenes?.length || 0);
      console.log('  - Animations 数量:', gltf.animations?.length || 0);
      console.log('  - Cameras 数量:', gltf.cameras?.length || 0);
      console.log('  - Asset 信息:', gltf.asset);
      
      if (gltf.animations && gltf.animations.length > 0) {
        console.log('\n🎬 动画信息:');
        gltf.animations.forEach((anim, i) => {
          console.log(`  [${i}] ${anim.name || '未命名动画'}`);
          console.log(`      持续时间: ${anim.duration}s`);
          console.log(`      轨道数: ${anim.tracks.length}`);
        });
      }
      
      // ===== 分析场景结构 =====
      console.log('\n========================================');
      console.log('🌳 场景层级结构:');
      console.log('========================================\n');
      
      let meshCount = 0;
      let materialSet = new Set();
      let textureSet = new Set();
      let totalVertices = 0;
      let totalFaces = 0;
      let meshesData = [];
      
      function printHierarchy(obj, level = 0) {
        const indent = '  '.repeat(level);
        const icon = obj.isMesh ? '🔷' : obj.isGroup ? '📁' : obj.isLight ? '💡' : obj.isCamera ? '📷' : '⚪';
        
        console.log(`${indent}${icon} ${obj.type}: "${obj.name || '未命名'}"`);
        
        if (obj.isMesh) {
          const geometry = obj.geometry;
          const material = obj.material;
          
          // 几何体信息
          const vertices = geometry.attributes.position?.count || 0;
          const faces = geometry.index ? geometry.index.count / 3 : vertices / 3;
          
          console.log(`${indent}  └─ 顶点数: ${vertices}, 面数: ${Math.floor(faces)}`);
          
          // 材质信息
          if (material) {
            if (Array.isArray(material)) {
              console.log(`${indent}  └─ 材质: 多材质 (${material.length}个)`);
              material.forEach(mat => materialSet.add(mat.uuid));
            } else {
              console.log(`${indent}  └─ 材质: ${material.type} "${material.name || '未命名'}"`);
              materialSet.add(material.uuid);
              
              // 打印材质详细信息
              printMaterialInfo(material, level + 2);
            }
          }
          
          // 位置信息
          const worldPos = new THREE.Vector3();
          obj.getWorldPosition(worldPos);
          console.log(`${indent}  └─ 世界坐标: (${worldPos.x.toFixed(2)}, ${worldPos.y.toFixed(2)}, ${worldPos.z.toFixed(2)})`);
          
          // 包围盒
          const box = new THREE.Box3().setFromObject(obj);
          const size = box.getSize(new THREE.Vector3());
          console.log(`${indent}  └─ 尺寸: ${size.x.toFixed(2)} × ${size.y.toFixed(2)} × ${size.z.toFixed(2)}`);
          
          meshCount++;
          totalVertices += vertices;
          totalFaces += faces;
          
          // 保存网格数据
          meshesData.push({
            name: obj.name,
            object: obj,
            vertices: vertices,
            faces: Math.floor(faces),
            position: worldPos,
            size: size
          });
        }
        
        // 递归处理子对象
        if (obj.children && obj.children.length > 0) {
          console.log(`${indent}  └─ 子对象数: ${obj.children.length}`);
          obj.children.forEach(child => printHierarchy(child, level + 1));
        }
      }
      
      function printMaterialInfo(material, level = 0) {
        const indent = '  '.repeat(level);
        
        console.log(`${indent}🎨 材质属性:`);
        console.log(`${indent}  - 类型: ${material.type}`);
        console.log(`${indent}  - 名称: ${material.name || '未命名'}`);
        console.log(`${indent}  - 颜色: ${material.color ? `#${material.color.getHexString()}` : 'N/A'}`);
        console.log(`${indent}  - 透明: ${material.transparent}`);
        console.log(`${indent}  - 不透明度: ${material.opacity}`);
        console.log(`${indent}  - 金属度: ${material.metalness ?? 'N/A'}`);
        console.log(`${indent}  - 粗糙度: ${material.roughness ?? 'N/A'}`);
        console.log(`${indent}  - 双面: ${material.side === THREE.DoubleSide ? '是' : '否'}`);
        
        // 纹理信息
        const textureProps = ['map', 'normalMap', 'roughnessMap', 'metalnessMap', 
                             'emissiveMap', 'aoMap', 'alphaMap', 'bumpMap', 
                             'displacementMap', 'envMap'];
        
        let hasTextures = false;
        textureProps.forEach(prop => {
          if (material[prop]) {
            if (!hasTextures) {
              console.log(`${indent}  🖼️ 纹理贴图:`);
              hasTextures = true;
            }
            
            const texture = material[prop];
            console.log(`${indent}    - ${prop}:`);
            console.log(`${indent}        UUID: ${texture.uuid}`);
            console.log(`${indent}        尺寸: ${texture.image?.width || '?'} × ${texture.image?.height || '?'}`);
            console.log(`${indent}        格式: ${texture.format}`);
            console.log(`${indent}        类型: ${texture.type}`);
            console.log(`${indent}        Source: ${texture.source?.data?.src || 'embedded'}`);
            
            textureSet.add(texture.uuid);
          }
        });
      }
      
      printHierarchy(gltf.scene);
      
      // ===== 统计信息 =====
      console.log('\n========================================');
      console.log('📊 模型统计信息:');
      console.log('========================================');
      console.log(`  🔷 网格总数: ${meshCount}`);
      console.log(`  🎨 材质总数: ${materialSet.size}`);
      console.log(`  🖼️ 纹理总数: ${textureSet.size}`);
      console.log(`  📐 总顶点数: ${totalVertices.toLocaleString()}`);
      console.log(`  🔺 总面数: ${Math.floor(totalFaces).toLocaleString()}`);
      console.log('========================================\n');
      
      // 更新UI信息
      modelInfo.value = {
        meshCount: meshCount,
        materialCount: materialSet.size,
        textureCount: textureSet.size,
        totalVertices: totalVertices,
        totalFaces: Math.floor(totalFaces),
        meshes: meshesData
      };
      
      // ===== 识别和匹配行星 =====
      console.log('\n========================================');
      console.log('🌍 识别和匹配行星:');
      console.log('========================================\n');
      
      gltf.scene.traverse((child) => {
        if (child.isMesh || child.isSkinnedMesh) {
          const material = child.material;
          if (material) {
            const materialName = material.name;
            
            // 根据材质名称匹配行星
            const planet = planets.find(p => p.materialName === materialName);
            if (planet) {
              console.log(`✓ 找到 ${planet.displayName} (材质: ${materialName})`);
              planetObjects[planet.name] = {
                mesh: child,
                config: planet
              };
            }
          }
        }
      });
      
      console.log('\n匹配结果:', Object.keys(planetObjects));
      
      // ===== 重新排列行星 =====
      arrangePlanets();
      
      // 更新显示的行星列表
      displayedPlanets.value = planets.map(p => ({
        ...p,
        object: planetObjects[p.name]?.displayMesh || null
      }));
      
      // ===== 打印所有材质详细信息 =====
      console.log('========================================');
      console.log('🎨 所有材质详细信息:');
      console.log('========================================\n');
      
      const materials = [];
      gltf.scene.traverse((obj) => {
        if (obj.isMesh && obj.material) {
          if (Array.isArray(obj.material)) {
            materials.push(...obj.material);
          } else {
            materials.push(obj.material);
          }
        }
      });
      
      // 去重
      const uniqueMaterials = [...new Map(materials.map(m => [m.uuid, m])).values()];
      
      uniqueMaterials.forEach((mat, index) => {
        console.log(`\n📦 材质 [${index + 1}/${uniqueMaterials.length}]:`);
        printMaterialInfo(mat, 1);
      });
      
      // ===== 打印所有纹理详细信息 =====
      console.log('\n========================================');
      console.log('🖼️ 所有纹理详细信息:');
      console.log('========================================\n');
      
      const textures = [];
      uniqueMaterials.forEach(mat => {
        const textureProps = ['map', 'normalMap', 'roughnessMap', 'metalnessMap', 
                             'emissiveMap', 'aoMap', 'alphaMap', 'bumpMap', 
                             'displacementMap', 'envMap'];
        textureProps.forEach(prop => {
          if (mat[prop]) textures.push({ type: prop, texture: mat[prop], material: mat.name });
        });
      });
      
      // 去重
      const uniqueTextures = [...new Map(textures.map(t => [t.texture.uuid, t])).values()];
      
      uniqueTextures.forEach((item, index) => {
        const tex = item.texture;
        console.log(`\n🖼️ 纹理 [${index + 1}/${uniqueTextures.length}]:`);
        console.log(`  - 类型: ${item.type}`);
        console.log(`  - 所属材质: ${item.material || '未命名'}`);
        console.log(`  - UUID: ${tex.uuid}`);
        console.log(`  - 尺寸: ${tex.image?.width || '?'} × ${tex.image?.height || '?'}`);
        console.log(`  - 格式: ${tex.format}`);
        console.log(`  - 数据类型: ${tex.type}`);
        console.log(`  - 编码: ${tex.encoding}`);
        console.log(`  - 过滤模式: minFilter=${tex.minFilter}, magFilter=${tex.magFilter}`);
        console.log(`  - 包裹模式: wrapS=${tex.wrapS}, wrapT=${tex.wrapT}`);
        console.log(`  - 各向异性: ${tex.anisotropy}`);
        console.log(`  - Source: ${tex.source?.data?.src || 'embedded data'}`);
      });
      
      // 添加原始模型到场景（已隐藏）
      scene.add(gltf.scene);
      
      loading.value = false;
      
      console.log('\n========================================');
      console.log('✅ 所有模型信息已打印完成！');
      console.log('========================================\n');
    },
    (progress) => {
      const percent = (progress.loaded / progress.total) * 100;
      loadingProgress.value = Math.round(percent);
      console.log(`⏳ 加载进度: ${loadingProgress.value}%`);
    },
    (error) => {
      console.error('\n❌ GLB 模型加载失败:', error);
      loading.value = false;
      alert('加载 GLB 模型失败，请检查文件路径和格式');
    }
  );
}

// 重新排列行星 - 太阳系布局
function arrangePlanets() {
  console.log('\n========================================');
  console.log('🎨 创建太阳系布局:');
  console.log('========================================\n');
  
  planets.forEach((planetConfig, index) => {
    const planetData = planetObjects[planetConfig.name];
    if (!planetData) {
      console.warn(`⚠️ 未找到 ${planetConfig.displayName}`);
      return;
    }
    
    const originalMesh = planetData.mesh;
    
    // 创建新的网格
    let newMesh;
    const geometry = originalMesh.geometry;
    const material = originalMesh.material;
    
    if (originalMesh.isSkinnedMesh) {
      const clonedGeometry = geometry.clone();
      clonedGeometry.computeVertexNormals();
      newMesh = new THREE.Mesh(clonedGeometry, material.clone());
    } else {
      const clonedGeometry = geometry.clone();
      clonedGeometry.computeVertexNormals();
      newMesh = new THREE.Mesh(clonedGeometry, material.clone());
    }
    
    // 设置缩放
    newMesh.scale.set(planetConfig.scale, planetConfig.scale, planetConfig.scale);
    
    // 调整所有行星的方向（统一绕X轴旋转90度）
    newMesh.rotation.x = Math.PI / 2;
    
    // 初始位置（行星会在轨道上，初始角度随机分布）
    const initialAngle = (index / planets.length) * Math.PI * 2;
    const x = Math.cos(initialAngle) * planetConfig.orbitRadius;
    const z = Math.sin(initialAngle) * planetConfig.orbitRadius;
    newMesh.position.set(x, 0, z);
    
    // 确保材质可见
    newMesh.visible = true;
    newMesh.castShadow = true;
    newMesh.receiveShadow = true;
    
    // 添加到场景
    scene.add(newMesh);
    planetMeshes.push(newMesh);
    
    // 保存配置信息
    planetObjects[planetConfig.name].displayMesh = newMesh;
    planetObjects[planetConfig.name].config = planetConfig;
    planetObjects[planetConfig.name].currentAngle = initialAngle; // 当前轨道角度
    
    // 如果是太阳，添加轨道线
    if (planetConfig.orbitRadius > 0) {
      const orbitGeometry = new THREE.BufferGeometry();
      const orbitPoints = [];
      for (let i = 0; i <= 64; i++) {
        const angle = (i / 64) * Math.PI * 2;
        orbitPoints.push(
          Math.cos(angle) * planetConfig.orbitRadius,
          0,
          Math.sin(angle) * planetConfig.orbitRadius
        );
      }
      orbitGeometry.setAttribute('position', new THREE.Float32BufferAttribute(orbitPoints, 3));
      const orbitMaterial = new THREE.LineBasicMaterial({ 
        color: 0x444444, 
        transparent: true, 
        opacity: 0.3 
      });
      const orbitLine = new THREE.Line(orbitGeometry, orbitMaterial);
      scene.add(orbitLine);
    }
    
    console.log(`  ${planetConfig.icon} ${planetConfig.displayName}:`);
    console.log(`     轨道半径: ${planetConfig.orbitRadius}`);
    console.log(`     缩放比例: ${planetConfig.scale}`);
    console.log(`     公转速度: ${planetConfig.orbitSpeed}`);
    console.log(`     自转速度: ${planetConfig.rotationSpeed}`);
  });
  
  // 调整相机位置以查看整个太阳系
  camera.position.set(0, 100, 150);
  controls.target.set(0, 0, 0);
  controls.minDistance = 0.1; // 可以拉得极近，观察纹理细节
  controls.maxDistance = 1000; // 可以拉得非常远，俯瞰整个太阳系
  controls.update();
  
  console.log('\n✓ 太阳系布局完成');
  console.log(`  相机位置: (0, 100, 150)`);
  console.log(`  已添加 ${planetMeshes.length} 个天体到场景`);
  console.log(`  太阳在中心 (0, 0, 0)`);
}

// 停止跟随并返回全景视图
function stopFollowing() {
  isFollowingPlanet.value = false;
  selectedPlanet.value = null;
  
  console.log('🌌 返回全景视图');
  
  // 平滑移动到全景位置
  gsap.to(camera.position, {
    x: 0,
    y: 100,
    z: 150,
    duration: 2,
    ease: "power2.inOut"
  });
  
  gsap.to(controls.target, {
    x: 0,
    y: 0,
    z: 0,
    duration: 2,
    ease: "power2.inOut"
  });
}

function focusOnPlanet(planet) {
  console.log('\n🎯 聚焦到:', planet.displayName);
  selectedPlanet.value = planet.name;
  isFollowingPlanet.value = true; // 开始跟随行星
  
  const planetData = planetObjects[planet.name];
  if (!planetData || !planetData.displayMesh) {
    console.error('❌ 未找到行星对象');
    return;
  }
  
  const mesh = planetData.displayMesh;
  const currentPosition = mesh.position.clone();
  const planetConfig = planetData.config;
  
  // 根据行星缩放比例计算统一的观察距离
  // 让所有行星在视觉上看起来大小相近
  const baseDistance = 15; // 基准距离
  const scaleFactor = 0.006; // 基准缩放（最大行星的缩放）
  const distance = baseDistance * (scaleFactor / planetConfig.scale);
  
  console.log('  行星缩放:', planetConfig.scale);
  console.log('  观察距离:', distance.toFixed(2));
  
  // 目标相机位置（在行星斜上方）
  const direction = currentPosition.clone().normalize();
  if (direction.length() === 0) {
    // 如果是太阳（在中心），使用默认方向
    direction.set(1, 0, 1).normalize();
  }
  
  const targetPosition = new THREE.Vector3(
    currentPosition.x + direction.x * distance * 0.7,
    currentPosition.y + distance * 0.5,
    currentPosition.z + direction.z * distance * 0.7
  );
  
  console.log('  目标位置:', currentPosition);
  console.log('  相机位置:', targetPosition);
  
  // 平滑移动相机
  gsap.to(camera.position, {
    x: targetPosition.x,
    y: targetPosition.y,
    z: targetPosition.z,
    duration: 2,
    ease: "power2.inOut"
  });
  
  // 设置控制器目标
  controls.target.copy(currentPosition);
  controls.update();
}

function animate() {
  animationId = requestAnimationFrame(animate);
  
  // 行星公转和自转
  planets.forEach((planetConfig) => {
    const planetData = planetObjects[planetConfig.name];
    if (!planetData || !planetData.displayMesh) return;
    
    const mesh = planetData.displayMesh;
    
    // 所有行星统一绕Z轴自转（因为已经旋转了90度）
    mesh.rotation.z += planetConfig.rotationSpeed;
    
    // 公转（如果不是太阳）
    if (planetConfig.orbitRadius > 0) {
      planetData.currentAngle += planetConfig.orbitSpeed * 0.01;
      
      const x = Math.cos(planetData.currentAngle) * planetConfig.orbitRadius;
      const z = Math.sin(planetData.currentAngle) * planetConfig.orbitRadius;
      
      mesh.position.set(x, 0, z);
      planetData.position = mesh.position.clone();
    }
  });
  
  // 如果正在跟随某个行星，相机要跟随它移动
  if (isFollowingPlanet.value && selectedPlanet.value) {
    const planetData = planetObjects[selectedPlanet.value];
    if (planetData && planetData.displayMesh) {
      const planetPosition = planetData.displayMesh.position.clone();
      const planetConfig = planetData.config;
      
      // 计算相机应该在的位置（保持相对位置）
      const baseDistance = 15;
      const scaleFactor = 0.006;
      const distance = baseDistance * (scaleFactor / planetConfig.scale);
      
      const direction = planetPosition.clone().normalize();
      if (direction.length() === 0) {
        direction.set(1, 0, 1).normalize();
      }
      
      const targetCameraPosition = new THREE.Vector3(
        planetPosition.x + direction.x * distance * 0.7,
        planetPosition.y + distance * 0.5,
        planetPosition.z + direction.z * distance * 0.7
      );
      
      // 平滑跟随（使用lerp插值）
      camera.position.lerp(targetCameraPosition, 0.05);
      
      // 控制器目标始终指向行星
      controls.target.copy(planetPosition);
    }
  }
  
  if (controls) controls.update();
  
  if (renderer && scene && camera) {
    renderer.render(scene, camera);
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
.glb-viewer-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  padding: 0;
}

.viewer-container {
  width: 100%;
  height: 100%;
  background-color: #1a1a2e;
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
  background: rgba(26, 26, 46, 0.95);
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

/* 信息面板 */
.info-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 350px;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(66, 184, 131, 0.3);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
  transition: all 0.3s ease;
  z-index: 100;
  max-height: 80vh;
  overflow: hidden;
}

.info-panel.collapsed {
  width: 60px;
}

.panel-header {
  padding: 15px 20px;
  background: rgba(66, 184, 131, 0.2);
  border-bottom: 1px solid rgba(66, 184, 131, 0.3);
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  user-select: none;
}

.info-panel.collapsed .panel-header {
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

.info-panel.collapsed .toggle-icon {
  display: none;
}

.panel-content {
  max-height: calc(80vh - 60px);
  overflow-y: auto;
  padding: 15px;
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

.info-section {
  margin-bottom: 20px;
}

.info-section h4 {
  color: #42b883;
  margin: 0 0 10px 0;
  font-size: 16px;
  border-bottom: 1px solid rgba(66, 184, 131, 0.3);
  padding-bottom: 5px;
}

.info-item {
  display: flex;
  justify-content: space-between;
  padding: 8px 0;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
}

.info-item .label {
  color: rgba(255, 255, 255, 0.6);
  font-size: 14px;
}

.info-item .value {
  color: white;
  font-size: 14px;
  font-weight: 600;
}

.planet-list {
  max-height: 400px;
  overflow-y: auto;
}

.planet-item {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  border: 2px solid transparent;
}

.planet-item:hover {
  background: rgba(66, 184, 131, 0.2);
  border-color: rgba(66, 184, 131, 0.5);
  transform: translateX(-5px);
}

.planet-item.active {
  background: rgba(66, 184, 131, 0.3);
  border-color: #42b883;
}

.planet-icon {
  font-size: 28px;
  margin-right: 12px;
  min-width: 35px;
  text-align: center;
}

.planet-info {
  flex: 1;
}

.planet-name {
  color: white;
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 4px;
}

.planet-status {
  font-size: 12px;
  color: #42b883;
}

.planet-status.missing {
  color: #ff6b6b;
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
  display: flex;
  align-items: center;
  gap: 15px;
}

.stop-follow-btn {
  background: #42b883;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  transition: all 0.3s ease;
  white-space: nowrap;
}

.stop-follow-btn:hover {
  background: #35a372;
  transform: scale(1.05);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .info-panel {
    width: 300px;
    top: 10px;
    right: 10px;
  }
  
  .controls-hint {
    font-size: 12px;
    padding: 8px 15px;
    bottom: 10px;
  }
}
</style>

