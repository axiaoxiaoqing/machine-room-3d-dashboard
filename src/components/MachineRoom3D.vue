<template>
  <div class="model-container">
    <div ref="containerRef"></div>
    <div 
      v-if="selectedObject && infoPosition" 
      class="info-panel"
      :style="{
        left: infoPosition.x + 'px',
        top: infoPosition.y + 'px'
      }"
    >
      <div class="info-header">
        <h3>{{ selectedObject.name || '未命名' }}</h3>
        <button @click="deselectObject" class="close-btn">×</button>
      </div>
      <div class="info-content">
        <p><strong>类型:</strong> {{ selectedObject.type }}</p>
        <p v-if="selectedObject.position">
          <strong>位置:</strong> 
          X:{{ selectedObject.position.x.toFixed(2) }}
          Y:{{ selectedObject.position.y.toFixed(2) }} 
          Z:{{ selectedObject.position.z.toFixed(2) }}
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'

const containerRef = ref(null)
const selectedObject = ref(null)
const infoPosition = ref(null)  // 信息框位置

let scene, camera, renderer, controls, raycaster, mouse
let highlightedMeshes = []
let highlightedWireframes = []
let originalMaterials = new Map()

onMounted(() => {
  init()
  loadModel()
})

onBeforeUnmount(() => {
  if (renderer) {
    renderer.dispose()
  }
  window.removeEventListener('resize', handleResize)
  window.removeEventListener('click', handleClick)
})

function init() {
  scene = new THREE.Scene()
  //改背景颜色
  scene.background = new THREE.Color('rgba(0, 32, 49, 0.9)')

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(10, 15, 15)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  
  const canvas = renderer.domElement
  canvas.style.width = '100%'
  canvas.style.height = '100%'
  canvas.style.display = 'block'
  containerRef.value.appendChild(canvas)

  controls = new OrbitControls(camera, canvas)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.target.set(0, 5, 0)

  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(10, 20, 0)
  scene.add(directionalLight)



  window.addEventListener('resize', handleResize)
  window.addEventListener('click', handleClick)
  
  animate()
}

function handleResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}


// 修改selectObject函数
function selectObject(group) {
  // 清除之前的高亮
  deselectObject()

  // 查找要高亮的所有Mesh对象
  const objectsToHighlight = []
  
  // 遍历group及其所有子对象，收集所有Mesh
  group.traverse((child) => {
    if (child.isMesh) {
      objectsToHighlight.push(child)
    }
  })
  
  // 如果没有Mesh，直接使用group本身
  if (objectsToHighlight.length === 0) {
    objectsToHighlight.push(group)
  }

  console.log('选中组对象:', group)
  console.log('要高亮的Mesh数量:', objectsToHighlight.length)

  // 为所有Mesh添加边框高亮
  objectsToHighlight.forEach(mesh => {
    highlightMesh(mesh)
  })
  
  // 更新选中对象信息
  selectedObject.value = {
    name: group.name || group.parent?.name || '未命名',
    type: group.type,
    position: group.position,
    uuid: group.uuid
  }
}

// 为单个Mesh添加高亮边框
function highlightMesh(mesh) {
  // 保存到高亮数组
  highlightedMeshes.push(mesh)
  
  // 保存原始材质
  if (!originalMaterials.has(mesh)) {
    originalMaterials.set(mesh, mesh.material)
  }
  
  // 创建边框几何体
  const edgesGeometry = new THREE.EdgesGeometry(mesh.geometry)
  const wireframeMaterial = new THREE.LineBasicMaterial({
    color: 0x00ff88, // 高亮颜色
    linewidth: 2
  })
  
  // 创建边框线框
  const wireframe = new THREE.LineSegments(edgesGeometry, wireframeMaterial)
  
  // 同步mesh的变换
  wireframe.position.copy(mesh.position)
  wireframe.rotation.copy(mesh.rotation)
  wireframe.scale.copy(mesh.scale)
  
  // 设置renderOrder确保边框显示在顶部
  wireframe.renderOrder = 1
  mesh.renderOrder = 0
  
  // 将边框添加到场景中（作为mesh的子对象）
  mesh.add(wireframe)
  
  // 添加到边框数组
  highlightedWireframes.push(wireframe)
}

// 修改deselectObject函数
function deselectObject() {
  // 移除所有高亮边框
  highlightedWireframes.forEach(wireframe => {
    if (wireframe.parent) {
      wireframe.parent.remove(wireframe)
    }
    // 清理几何体和材质
    wireframe.geometry.dispose()
    wireframe.material.dispose()
  })
  
  // 恢复原始材质（如果需要）
  highlightedMeshes.forEach(mesh => {
    if (originalMaterials.has(mesh)) {
      // 如果你修改了mesh的材质，可以在这里恢复
      // mesh.material = originalMaterials.get(mesh)
      originalMaterials.delete(mesh)
    }
  })
  
  // 清空数组
  highlightedMeshes.length = 0
  highlightedWireframes.length = 0
  
  // 重置所有状态
  selectedObject.value = null
  infoPosition.value = null  // 清除信息框位置
  
  console.log('取消选择，清理高亮边框')
}



// 修改handleClick函数中的选择逻辑
function handleClick(event) {
  deselectObject()
  
  const rect = renderer.domElement.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(scene.children, true)

  if (intersects.length > 0) {
    // 查找带有"机柜"的对象
    let targetObj = null
    
    for (const intersect of intersects) {
      let obj = intersect.object
      
      // 检查当前对象及其所有父对象的名称
      while (obj) {
        if (obj.name && obj.name.includes('机柜')) {
          targetObj = obj
          break
        }
        obj = obj.parent
      }
      
      if (targetObj) break
    }
    
    if (targetObj) {
      // 执行选择
      selectObject(targetObj)
      
      // 设置信息面板位置
      setInfoPanelPosition(event)
    } else {
      deselectObject()
    }
  } else {
    deselectObject()
  }
}

// 添加设置信息面板位置的函数
function setInfoPanelPosition(event) {
  infoPosition.value = {
    x: event.clientX + 20,  // 向右偏移20px
    y: event.clientY - 20   // 向上偏移20px
  }
}




function loadModel() {
  const loader = new GLTFLoader()
  const textureLoader = new THREE.TextureLoader()
  const cabinetTexture = textureLoader.load('/models/cabinet.jpg')
  
  loader.load(
    '/models/myjifang.glb',
    (gltf) => {
      const model = gltf.scene
      
      // 为模型各部分命名以便识别
      model.traverse((child, index) => {
        if (child.isMesh) {
          if (!child.name) {
            child.name = `设备部件_${index + 1}`
          }
          
        
          
          if (child.material) {
            // 为机柜添加纹理
            if (child.name.toLowerCase().includes('cabinet') || 
                child.name.toLowerCase().includes('机柜')) {
              child.material = new THREE.MeshPhongMaterial({
                map: cabinetTexture,
                shininess: 30
              })
            } else {
              child.material.metalness = 0.2
              child.material.roughness = 0.8
            }
          }
        }
      })
      
      
      const box = new THREE.Box3().setFromObject(model)
      const center = box.getCenter(new THREE.Vector3())
      const size = box.getSize(new THREE.Vector3())
      
      model.position.x = -center.x
      model.position.y = -center.y
      model.position.z = -center.z
      
      const maxDim = Math.max(size.x, size.y, size.z)
      const scale = 40 / maxDim
      model.scale.setScalar(scale)
      
      scene.add(model)
      
      camera.position.set(0, size.y * 1.8, size.z * 2.2)
      controls.target.set(0, size.y * 0.6, 0)
      controls.update()
      
      console.log('模型加载成功')
    },
    (progress) => {
      console.log('加载进度:', (progress.loaded / progress.total * 100).toFixed(2) + '%')
    },
    (error) => {
      console.error('加载模型失败:', error)
      createDefaultModel()
    }
  )
}

// 创建默认模型函数
function createDefaultModel() {
  const textureLoader = new THREE.TextureLoader()
  const cabinetTexture = textureLoader.load('/models/cabinet.jpg')
  
  // 创建地面
  const floorGeometry = new THREE.PlaneGeometry(50, 50)
  const floorMaterial = new THREE.MeshPhongMaterial({ 
    color: 0x444444,
    side: THREE.DoubleSide,
    metalness: 0.1,
    roughness: 0.8
  })
  const floor = new THREE.Mesh(floorGeometry, floorMaterial)
  floor.rotation.x = -Math.PI / 2
  floor.position.y = 0
  floor.name = 'floor'
  scene.add(floor)
  
  // 创建几个机柜作为示例
  for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3; j++) {
      // 创建机柜主体
      const cabinetGeometry = new THREE.BoxGeometry(3, 8, 1.5)
      const cabinetMaterial = new THREE.MeshPhongMaterial({
        map: cabinetTexture,
        shininess: 30
      })
      const cabinet = new THREE.Mesh(cabinetGeometry, cabinetMaterial)
      cabinet.position.set((i - 1) * 5, 4, (j - 1) * 5)
      cabinet.name = `body${String(i * 3 + j).padStart(3, '0')}` // body000, body001等
      cabinet.userData.isSelectable = true
      scene.add(cabinet)
      
      // 创建机柜门
      const doorGeometry = new THREE.BoxGeometry(2.8, 7.6, 0.1)
      const doorMaterial = new THREE.MeshPhongMaterial({
        color: 0x666666,
        shininess: 20
      })
      const door = new THREE.Mesh(doorGeometry, doorMaterial)
      door.position.set((i - 1) * 5, 4, (j - 1) * 5 + 0.75)
      door.name = `door-body${String(i * 3 + j).padStart(3, '0')}` // door-body000, door-body001等
      door.userData.isSelectable = true
      scene.add(door)
    }
  }
  
  // 设置相机位置
  camera.position.set(10, 15, 15)
  controls.target.set(0, 5, 0)
  controls.update()
  
  console.log('默认模型创建成功')
}

function animate() {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}
</script>

<style scoped>
.model-container {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  position: fixed;
  top: 0;
  left: 0;
  overflow: hidden;
  background-color: #000;
}

.model-container > div:first-child {
  width: 100%;
  height: 100%;
}

.info-panel {
  position: fixed;
  background: rgba(30, 41, 59, 0.95);
  color: white;
  padding: 12px;
  border-radius: 8px;
  border: 2px solid #2196f3;
  min-width: 220px;
  max-width: 280px;
  backdrop-filter: blur(10px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
  z-index: 1000;
  transform: translateY(-50%);
  animation: fadeIn 0.2s ease;
}

.info-panel::before {
  content: '';
  position: absolute;
  left: -8px;
  top: 50%;
  transform: translateY(-50%);
  border-right: 8px solid #2196f3;
  border-top: 6px solid transparent;
  border-bottom: 6px solid transparent;
}

.info-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
}

.info-header h3 {
  margin: 0;
  color: #2196f3;
  font-size: 16px;
  font-weight: 600;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.close-btn {
  background: rgba(255, 68, 68, 0.2);
  color: #51ff44;
  border: 1px solid rgba(255, 68, 68, 0.3);
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
  margin-left: 10px;
  transition: all 0.3s;
}

.close-btn:hover {
  background: rgba(255, 68, 68, 0.3);
  border-color: rgba(255, 68, 68, 0.5);
}

.info-content {
  font-size: 14px;
}

.info-content p {
  margin: 6px 0;
  line-height: 1.4;
}

.info-content strong {
  color: #94a3b8;
  margin-right: 6px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-50%) translateX(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(-50%) translateX(0);
  }
}

/* 确保信息框不会超出屏幕边界 */
.info-panel {
  max-width: calc(100vw - 40px);
}

/* 如果信息框在屏幕右侧，箭头移到左边 */
.info-panel.right-side::before {
  left: auto;
  right: -8px;
  border-right: none;
  border-left: 8px solid #2196f3;
}

* {
  box-sizing: border-box;
}

body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
</style>