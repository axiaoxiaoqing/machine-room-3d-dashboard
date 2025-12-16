<template>
  <div id="container" @mousemove="mouseMove">
    <div
      id='plane'
      :style="{left: state.planePos.left, top: state.planePos.top, display: state.planeDisplay}"
    >
      <p>机柜名称：{{ state.curCabinet.name }}</p>
      <p>机柜温度：{{ state.curCabinet.temperature.toFixed(1) }}°</p>
      <p>使用情况：{{ state.curCabinet.count}} / {{ state.curCabinet.capacity}}</p>
    </div>
  </div>
</template>

<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'

// 定义事件
const emit = defineEmits(['cabinet-selected', 'temperature-changed', 'alarm-generated'])

// 基础3D场景对象
let mesh = null
let camera = null
let scene = null
let renderer = null
let controls = null
let maps = null
let temperatureMonitorTimer = null

// 响应式数据
const cabinets = ref([])
const curCabinet = ref(null)

const state = reactive({
  planePos: {
    left: 0,
    top: 0
  },
  planeDisplay: 'none',
  curCabinet: {
    name: 'Loading……',
    temperature: 0,
    capacity: 0,
    count: 0
  }
})

// 初始化
function init() {
  createScene() // 创建场景
  loadGLTF() // 加载GLTF模型
  // createLight() // 创建光源
  createCamera() // 创建相机
  createRender() // 创建渲染器
  createControls() // 创建控件对象
  render() // 渲染
}

// 创建场景
function createScene() {
  scene = new THREE.Scene()
}

// 加载GLTF模型
function loadGLTF() {
  const loader = new GLTFLoader()
  loader.load(`./models/machineRoom.gltf`, ({ scene: { children } }) => {
    console.log(...children);
    children.forEach((obj) => {
      const { map, color } = obj.material
      changeMat(obj, map, color)
      if (obj.name.includes('cabinet')) {
        cabinets.value.push(obj)
      }
    })

    scene.add(...children)
  })
}

// 创建光源
function createLight() {
  // 环境光
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.1) // 创建环境光
  scene.add(ambientLight) // 将环境光添加到场景

  const spotLight = new THREE.SpotLight(0xffffff) // 创建聚光灯
  spotLight.position.set(150, 150, 150)
  spotLight.castShadow = true
  scene.add(spotLight)
}

// 创建相机
function createCamera() {
  const element = document.getElementById('container')
  const width = element.clientWidth // 窗口宽度
  const height = element.clientHeight // 窗口高度
  const k = width / height // 窗口宽高比
  // PerspectiveCamera( fov, aspect, near, far )
  camera = new THREE.PerspectiveCamera(45, k, 0.1, 1000)
  camera.position.set(0, 10, 15) // 设置相机位置

  camera.lookAt(0, 0, 0) // 设置相机方向
  scene.add(camera)
}

// 创建渲染器
function createRender() {
  const element = document.getElementById('container')
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setSize(element.clientWidth, element.clientHeight) // 设置渲染区域尺寸
  // renderer.shadowMap.enabled = true // 显示阴影
  // renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setClearColor(0x3f3f3f, 1) // 设置背景颜色
  element.appendChild(renderer.domElement)
}

function render() {
  // if (mesh) {
  //   mesh.rotation.z += 0.006
  // }
  renderer.render(scene, camera)
  requestAnimationFrame(render)
}

// 创建控件对象
function createControls() {
  controls = new OrbitControls(camera, renderer.domElement)
}

/**
 * obj：需要修改材质的Mesh 对象
 * map：GLTF 模型里的贴图对象
 * color：GLTF 模型的颜色
 */
function changeMat(obj, map, color) {
  if (map) {
    obj.material = new THREE.MeshBasicMaterial({
      map: crtTexture(map.name)
    })
  } else {
    obj.material = new THREE.MeshBasicMaterial({ color })
  }
}

function crtTexture(imgName) {
  let curTexture = maps.get(imgName)
  if (!curTexture) {
    curTexture = new THREE.TextureLoader().load('./models/' + imgName)
    curTexture.flipY = false
    curTexture.wrapS = 1000
    curTexture.wrapT = 1000
    maps.set(
      imgName,
      curTexture
    )
  }
  return curTexture
}

function selectCabinet(x, y) {
  const { width, height } = renderer.domElement
  //射线投射器，可基于鼠标点和相机，在世界坐标系内建立一条射线，用于选中模型
  const raycaster = new THREE.Raycaster()
  //鼠标在裁剪空间中的点位
  const pointer = new THREE.Vector2()

  // 鼠标的canvas坐标转裁剪坐标
  pointer.set(
    (x / width) * 2 - 1,
    -(y / height) * 2 + 1,
  )
  // 基于鼠标点的裁剪坐标位和相机设置射线投射器
  raycaster.setFromCamera(
    pointer, camera
  )
  // 选择机柜
  const intersect = raycaster.intersectObjects(cabinets.value)[0]
  let intersectObj = intersect ? intersect.object : null
  // 若之前已有机柜被选择，且不等于当前所选择的机柜，取消之前选择的机柜的高亮
  if (curCabinet.value && curCabinet.value !== intersectObj) {
    const material = curCabinet.value.material
    material.setValues({
      map: maps.get('cabinet.jpg')
    })
  }
  /* 
    若当前所选对象不为空：
      触发鼠标在机柜上移动的事件。
      若当前所选对象不等于上一次所选对象：
        更新curCabinet。
        将模型高亮。
        触发鼠标划入机柜事件。
    否则若上一次所选对象存在：
      置空curCabinet。
      触发鼠标划出机柜事件。
  */
  if (intersectObj) {
    onMouseMoveCabinet(x, y)
    if (intersectObj !== curCabinet.value) {
      curCabinet.value = intersectObj
      const material = intersectObj.material
      material.setValues({
        map: maps.get('cabinet-hover.jpg')
      })
      onMouseOverCabinet(intersectObj)
    }
  } else if (curCabinet.value) {
    curCabinet.value = null
    onMouseOutCabinet()
  }
}

// 鼠标移动事件
function mouseMove({ clientX, clientY }) {
  selectCabinet(clientX, clientY)
}

//鼠标划入机柜事件，参数为机柜对象
function onMouseOverCabinet(cabinet) {
  console.log(cabinet.name)
  state.curCabinet.name = cabinet.name
  state.planeDisplay = 'block'
  
  // 触发机柜选中事件
  emit('cabinet-selected', {
    name: cabinet.name,
    temperature: state.curCabinet.temperature,
    capacity: state.curCabinet.capacity,
    count: state.curCabinet.count
  })
  
  //基于cabinet.name 获取机柜数据
  // getCabinateByName(cabinet.name).then(({ data }) => {
  //   state.curCabinet = { ...data, name: cabinet.name }
  // });
}

//鼠标在机柜上移动的事件，参数为鼠标在canvas画布上的坐标位
function onMouseMoveCabinet(x, y) {
  // console.log(x,y);
  state.planePos.left = x + 'px'
  state.planePos.top = y + 'px'
}

//鼠标划出机柜的事件
function onMouseOutCabinet() {
  state.planeDisplay = 'none'
  emit('cabinet-selected', null)
}

// 更新温度数据
function updateTemperatureData() {
  // 模拟机柜温度变化
  if (curCabinet.value && state.curCabinet.name !== 'Loading……') {
    // 随机生成温度变化 (-1 到 +1 度)
    const tempChange = (Math.random() - 0.5) * 2
    let newTemp = state.curCabinet.temperature + tempChange

    // 限制温度范围在 18-45°C 之间
    newTemp = Math.max(18, Math.min(45, newTemp))
    state.curCabinet.temperature = Math.round(newTemp * 10) / 10

    // 触发温度变化事件
    emit('temperature-changed', {
      cabinetName: state.curCabinet.name,
      temperature: state.curCabinet.temperature
    })

    // 检查温度是否超过阈值，生成告警
    if (newTemp > 40) {
      generateAlarm(state.curCabinet.name, newTemp, 'high')
    } else if (newTemp > 35) {
      generateAlarm(state.curCabinet.name, newTemp, 'medium')
    }
  }
}

// 生成告警
function generateAlarm(cabinetName, temperature, level) {
  const now = new Date()
  const timeStr = now.toLocaleString('zh-CN')
  // 确保温度显示小数点后一位
  const formattedTemp = temperature.toFixed(1)
  const content = `机柜${cabinetName}温度过高（${formattedTemp}°C）`

  // 触发告警生成事件
  emit('alarm-generated', {
    id: Date.now(),
    time: timeStr,
    content: content,
    level: level
  })
}

// 生命周期钩子
onMounted(() => {
  maps = new Map()//添加maps属性，用来存储纹理对象，以避免贴图的重复加载
  crtTexture("cabinet-hover.jpg")
  init()

  // 启动温度监控定时器，每3秒更新一次数据
  temperatureMonitorTimer = setInterval(() => {
    updateTemperatureData()
  }, 3000)
})

onBeforeUnmount(() => {
  // 清除定时器
  if (temperatureMonitorTimer) {
    clearInterval(temperatureMonitorTimer)
  }
})
</script>

<style scoped>
#container {
  position: absolute;
  width: 100%;
  height: 100%;
}

#plane {
  position: absolute;
  top: 0;
  left: 0;
  background-color: rgba(0,0,0,0.5);
  color: #fff;
  padding: 0 18px;
  transform: translate(12px,-100%);
  display: none;
  text-align: left;
}
</style>