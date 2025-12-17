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
let highlightedMesh = null
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

  // 网格
  // const gridHelper = new THREE.GridHelper(50, 50)
  // gridHelper.material.opacity = 0.2
  // gridHelper.material.transparent = true
  // scene.add(gridHelper)

  window.addEventListener('resize', handleResize)
  window.addEventListener('click', handleClick)
  
  animate()
}

function handleResize() {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

function handleClick(event) {
  const rect = renderer.domElement.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(mouse, camera)
  const intersects = raycaster.intersectObjects(scene.children, true)

  // 过滤出可选择的几何体，只有名称为"body+编号"格式的对象才能被选中
  const filteredIntersects = intersects.filter(intersect => {
    let obj = intersect.object
    // 查找最外层的Mesh
    while (obj && !obj.isMesh) {
      obj = obj.parent
    }
    // 只选择名称为"body+编号"格式的对象（如body004）
    return obj && obj.isMesh && 
           /^body\d+$/.test(obj.name.toLowerCase())
  })

  if (filteredIntersects.length > 0) {
    let selectedMesh = filteredIntersects[0].object
    while (selectedMesh && !selectedMesh.isMesh) {
      selectedMesh = selectedMesh.parent
    }

    if (selectedMesh && selectedMesh.isMesh) {
      // 设置信息框位置为鼠标点击位置
      infoPosition.value = {
        x: event.clientX + 20,  // 向右偏移20px
        y: event.clientY - 20   // 向上偏移20px
      }
      selectObject(selectedMesh)
    } else {
      deselectObject()
    }
  } else {
    deselectObject()
  }
}
let highlightedWireframe = null
const originalWireframes = new Map()
function selectObject(mesh) {
  // 如果已经选中相同的对象，则取消选择
  if (highlightedMesh === mesh) {
    deselectObject()
    return
  }

  // 清除之前的高亮
  if (highlightedMesh) {
    deselectObject()
  }

  highlightedMesh = mesh
  
  // 确保原始材质被保存
  if (!originalMaterials.has(mesh)) {
    originalMaterials.set(mesh, mesh.material)
  }

  // 创建边框线框
  const edgesGeometry = new THREE.EdgesGeometry(mesh.geometry)
  
  // 方案A：直接使用网格的几何体
  const wireframe = new THREE.LineSegments(
    edgesGeometry,
    new THREE.LineBasicMaterial({
      color: 0x00ff88,
      linewidth: 2
    })
  )
  
  // 关键：将线框的变换与网格完全同步
  wireframe.position.set(0, 0, 0)
  wireframe.rotation.set(0, 0, 0)
  wireframe.scale.set(1, 1, 1)
  
  // 将线框添加为网格的子对象，这样它会继承所有变换
  mesh.add(wireframe)
  
  // 更新世界矩阵
  mesh.updateMatrixWorld(true)
  
  highlightedWireframe = wireframe
  
  selectedObject.value = {
    name: mesh.name || mesh.parent?.name || '未命名',
    type: mesh.type,
    position: mesh.position,
    uuid: mesh.uuid
  }
}

// 同时需要修改取消选择函数
function deselectObject() {
  if (highlightedMesh) {
    // 移除高亮线框
    if (highlightedWireframe) {
      highlightedMesh.remove(highlightedWireframe)
      highlightedWireframe.geometry.dispose()
      highlightedWireframe.material.dispose()
      highlightedWireframe = null
    }
    
    // 恢复原始材质（如果需要的话）
    if (originalMaterials.has(highlightedMesh)) {
      // 如果你还想保持原始材质，可以取消注释下面这行
      // highlightedMesh.material = originalMaterials.get(highlightedMesh)
      originalMaterials.delete(highlightedMesh)
    }
    
    // 清理线框映射表中的数据
    if (originalWireframes.has(highlightedMesh)) {
      originalWireframes.delete(highlightedMesh)
    }
    
    highlightedMesh = null
  }
  selectedObject.value = null
}




function loadModel() {
  const loader = new GLTFLoader()
  const textureLoader = new THREE.TextureLoader()
  const cabinetTexture = textureLoader.load('/models/cabinet.jpg')
  
  loader.load(
    '/models/jifang3.glb',
    (gltf) => {
      const model = gltf.scene
      
      // 为模型各部分命名以便识别
      model.traverse((child, index) => {
        if (child.isMesh) {
          if (!child.name) {
            child.name = `设备部件_${index + 1}`
          }
          
          // 检查是否为地板，如果是则设置为不可选择
          const isFloor = child.name.toLowerCase().includes('floor') || 
                         child.name.toLowerCase().includes('地面') || 
                         child.name.toLowerCase().includes('地板')|| 
                         child.name.toLowerCase().includes('glass')
          
          if (isFloor) {
            child.userData.isSelectable = false
          } else {
            child.userData.isSelectable = true
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