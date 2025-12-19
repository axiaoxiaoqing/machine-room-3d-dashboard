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
    // 检查整个父链，直到找到名称为body+编号格式的对象
    while (obj) {
      if (obj.isMesh && /^body\d+$/.test(obj.name.toLowerCase())) {
        return true
      }
      obj = obj.parent
    }
    return false
  })

  if (filteredIntersects.length > 0) {
    // 从相交点中找到最外层的body对象
    let intersect = filteredIntersects[0]
    let selectedMesh = intersect.object
    
    // 向上查找，直到找到名称为body+编号格式的Mesh对象
    let bodyMesh = null
    let tempObj = selectedMesh
    while (tempObj) {
      if (tempObj.isMesh && /^body\d+$/.test(tempObj.name.toLowerCase())) {
        bodyMesh = tempObj
        break
      }
      tempObj = tempObj.parent
    }
    
    // 如果找到了body对象
    if (bodyMesh) {
      // 先调用selectObject（内部会调用deselectObject清除之前的高亮）
      selectObject(bodyMesh)
      // 然后再设置信息框位置，确保不会被deselectObject重置
      infoPosition.value = {
        x: event.clientX + 20,  // 向右偏移20px
        y: event.clientY - 20   // 向上偏移20px
      }
    } else {
      deselectObject()
    }
  } else {
    deselectObject()
  }
}
const originalWireframes = new Map()
function selectObject(mesh) {
  // 清除之前的高亮
  deselectObject()

  // 查找要高亮的所有对象：当前选中的body和对应的door-body
  const objectsToHighlight = [mesh]
  
  // 如果当前是body对象，尝试找到对应的door-body对象
  if (mesh.name && mesh.name.toLowerCase().startsWith('body')) {
    // 从body名称中提取编号，例如body004 → 004
    const bodyNumber = mesh.name.substring(4)  // 假设名称格式为"bodyXXX"
    const doorBodyName = `door-body${bodyNumber}`
    
    // 在场景中查找对应的door-body对象
    let doorBodyFound = false
    scene.traverse((child) => {
      if (child.isMesh && child.name === doorBodyName) {
        objectsToHighlight.push(child)
        doorBodyFound = true
      }
    })
    
    if (!doorBodyFound) {
      // 尝试另一种命名格式，例如body004 → doorbody004
      const alternativeDoorBodyName = `doorbody${bodyNumber}`
      scene.traverse((child) => {
        if (child.isMesh && child.name === alternativeDoorBodyName) {
          objectsToHighlight.push(child)
        }
      })
    }
  }

  // 为所有要高亮的对象创建边框
  objectsToHighlight.forEach(obj => {
    highlightedMeshes.push(obj)
    
    // 确保原始材质被保存
    if (!originalMaterials.has(obj)) {
      originalMaterials.set(obj, obj.material)
    }

    // 创建边框线框
    const edgesGeometry = new THREE.EdgesGeometry(obj.geometry)
    
    // 直接使用网格的几何体
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
    obj.add(wireframe)
    
    // 更新世界矩阵
    obj.updateMatrixWorld(true)
    
    highlightedWireframes.push(wireframe)
  })
  
  selectedObject.value = {
    name: mesh.name || mesh.parent?.name || '未命名',
    type: mesh.type,
    position: mesh.position,
    uuid: mesh.uuid
  }
}

// 同时需要修改取消选择函数
function deselectObject() {
  // 移除所有高亮线框
  highlightedWireframes.forEach(wireframe => {
    // 查找线框的父对象并移除
    if (wireframe.parent) {
      wireframe.parent.remove(wireframe)
    }
    wireframe.geometry.dispose()
    wireframe.material.dispose()
  })
  
  // 恢复所有原始材质（如果需要的话）
  highlightedMeshes.forEach(mesh => {
    if (originalMaterials.has(mesh)) {
      // 如果你还想保持原始材质，可以取消注释下面这行
      // mesh.material = originalMaterials.get(mesh)
      originalMaterials.delete(mesh)
    }
    
    // 清理线框映射表中的数据
    if (originalWireframes.has(mesh)) {
      originalWireframes.delete(mesh)
    }
  })
  
  // 清空数组 - 使用空数组重新赋值，确保引用也更新
  highlightedMeshes.length = 0
  highlightedWireframes.length = 0
  
  // 重置所有状态
  selectedObject.value = null
  infoPosition.value = null  // 清除信息框位置
}




function loadModel() {
  const loader = new GLTFLoader()
  const textureLoader = new THREE.TextureLoader()
  const cabinetTexture = textureLoader.load('/models/cabinet.jpg')
  
  loader.load(
    '/models/datacenter.glb',
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