<template>
    <div>
      <!-- 页面标题 -->
      <div id="page-title" class="header">
        智慧院数字孪生机房系统
      </div>
      
      <!-- 全局告警信息面板 -->
      <div id="alarm-panel" class="panel">
        <h3>告警信息</h3>
        <div class="alarm-list">
          <div v-for="alarm in alarms" :key="alarm.id" class="alarm-item" :class="alarm.level">
            <span class="alarm-time">{{ alarm.time }}</span>
            <span class="alarm-content">{{ alarm.content }}</span>
          </div>
          <div v-if="alarms.length === 0" class="no-alarms">暂无告警信息</div>
        </div>
      </div>
      
      <!-- 机房整体监控面板 -->
      <div id="monitor-panel" class="panel">
        <h3>机房监控</h3>
        <div class="monitor-grid">
          <div class="monitor-item">
            <span class="monitor-label">机房温度1</span>
            <span class="monitor-value">{{ roomMonitor.temperature.toFixed(1) }}°C</span>
          </div>
          <div class="monitor-item">
            <span class="monitor-label">机房温度2</span>
            <span class="monitor-value">{{ roomMonitor.temperature2.toFixed(1) }}%</span>
          </div>
        </div>
      </div>
      
      <!-- 视频监控面板 -->
      <div id="video-panel-1" class="video-panel panel" >
        <h3>视频监控 1</h3>
        <div class="video-placeholder">
          视频监控画面占位
        </div>
      </div>
      
      <div id="video-panel-2" class="video-panel panel">
        <h3>视频监控 2</h3>
        <div class="video-placeholder">
          视频监控画面占位
        </div>
      </div>
      
      <!-- 引入3D组件 -->
      <MachineRoom3D 
        @cabinet-selected="handleCabinetSelected" 
        @temperature-changed="handleTemperatureChange" 
        @alarm-generated="handleAlarmGenerated" 
      />
    </div>
  </template>
  
  <script setup>
  import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'
  import MachineRoom3D from './MachineRoom3D.vue'

  // 响应式数据
  const alarms = ref([
    { id: 1, time: '2024-03-07 10:15:30', content: '机柜cabinet_01温度过高（45.0°C）', level: 'high' },
    { id: 2, time: '2024-03-07 10:10:20', content: '机房湿度低于阈值（30%）', level: 'medium' },
    { id: 3, time: '2024-03-07 10:05:10', content: '机柜cabinet_05电力负载异常', level: 'low' }
  ])

  const roomMonitor = reactive({
    temperature: 28.0,
    temperature2: 28.1,



  })

  // 事件处理函数
  function handleCabinetSelected(cabinet) {
    console.log('机柜选中:', cabinet)
    // 可以在这里处理机柜选中事件
  }

  function handleTemperatureChange(data) {
    console.log('温度变化:', data)
    // 可以在这里处理温度变化事件
  }

  function handleAlarmGenerated(alarm) {
    console.log('生成告警:', alarm)
    
    // 检查是否已有相同内容的告警，避免重复
    const existingAlarm = alarms.value.find(existing =>
      existing.content === alarm.content &&
      Date.now() - new Date(existing.time).getTime() < 60000 // 1分钟内不重复
    )

    if (!existingAlarm) {
      // 添加到告警列表开头
      alarms.value.unshift(alarm)

      // 限制告警数量，最多显示10条
      if (alarms.value.length > 10) {
        alarms.value.pop()
      }
    }
  }

  // 机房监控数据更新定时器
  let roomMonitorTimer = null

  // 更新机房整体监控数据
  function updateRoomMonitorData() {
    // 更新机房整体温度
    const tempChange = (Math.random() - 0.5) * 1
    let newRoomTemp = roomMonitor.temperature + tempChange
    newRoomTemp = Math.max(20, Math.min(32, newRoomTemp))
    roomMonitor.temperature = Math.round(newRoomTemp * 10) / 10

    // 更新其他机房监控数据
    roomMonitor.humidity = Math.max(30, Math.min(60, roomMonitor.humidity + (Math.random() - 0.5) * 2))
    roomMonitor.powerLoad = Math.max(50, Math.min(90, roomMonitor.powerLoad + (Math.random() - 0.5) * 3))
  }

  // 生命周期钩子
  onMounted(() => {
    // 启动机房监控数据更新定时器，每3秒更新一次数据
    roomMonitorTimer = setInterval(() => {
      updateRoomMonitorData()
    }, 3000)
  })

  onBeforeUnmount(() => {
    // 清除定时器
    if (roomMonitorTimer) {
      clearInterval(roomMonitorTimer)
    }
  })
</script>
  <style>
  /* 页面标题样式 */
  #page-title {
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    font-size: 24px;
    font-weight: bold;
    color: #ffffff;

    padding: 10px 30px;
    z-index: 1001;
  }
  .header {
  position: absolute;
  top: 0;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100vw;
  /* height: 1rem; */
  /* padding-bottom: 10px; */
  font-size: 0.4rem;
  font-weight: bold;
  color: #fff;
  background: url('/public/models/header_bg.png');
  background-repeat: no-repeat;
  background-size: 100% 100%;
}

  .panel{
  padding: 20px;
  position: relative;
  box-sizing: border-box;
  /* min-width: 300px; */
  /* width: 450px; */
  /* padding: 0.1rem 0.2rem; */
  background: linear-gradient(#99fffe, #99fffe) left -3px top 0, linear-gradient(#99fffe, #99fffe) left -3px top -3px, linear-gradient(#99fffe, #99fffe) right -3px top 0, linear-gradient(#99fffe, #99fffe) right -3px top -3px, linear-gradient(#99fffe, #99fffe) left -3px bottom 0, linear-gradient(#99fffe, #99fffe) left -3px bottom -3px, linear-gradient(#99fffe, #99fffe) right -3px bottom 0, linear-gradient(#99fffe, #99fffe) right -3px bottom -3px;
  background-color: rgba(0, 34, 51, 0.6);
  background-repeat: no-repeat;
  background-size: 20px 20px, 20px 20px;
  background-size: 3px 16px, 16px 3px;
  border: 1px solid transparent;
  backdrop-filter: blur(1px);
  color: #fff;
  z-index: 2;
  }
  
  /* 告警信息面板样式 */
  #alarm-panel {
    position: absolute;
    top: 260px;
    left: 20px;
    width: 350px;
    color: #fff;
    padding: 15px;
    z-index: 1000;
  }
  
  #alarm-panel h3 {
    margin-top: 0;
    margin-bottom: 15px;
    color: #ff4444;
    font-size: 18px;
    border-bottom: 1px solid #ff4444;
    padding-bottom: 8px;
  }
  
  .alarm-list {
    max-height: 250px;
    overflow-y: auto;
  }
  
  .alarm-item {
    padding: 10px;
    margin-bottom: 8px;
    border-radius: 4px;
    font-size: 14px;
  }
  
  .alarm-item.high {
    background-color: rgba(255, 68, 68, 0.3);
    border-left: 4px solid #ff4444;
  }
  
  .alarm-item.medium {
    background-color: rgba(255, 165, 0, 0.3);
    border-left: 4px solid #ffa500;
  }
  
  .alarm-item.low {
    background-color: rgba(255, 235, 59, 0.3);
    border-left: 4px solid #ffeb3b;
  }
  
  .alarm-time {
    display: block;
    font-size: 12px;
    opacity: 0.8;
    margin-bottom: 4px;
  }
  
  .alarm-content {
    display: block;
  }
  
  .no-alarms {
    text-align: center;
    color: #888;
    padding: 20px;
  }
  
  /* 机房监控面板样式 */
  #monitor-panel {
    position: absolute;
    top: 60px;
    left: 20px;
    width: 350px;
    color: #fff;
    padding: 15px;
    z-index: 1000;
  }
  
  #monitor-panel h3 {
    margin-top: 0;
    margin-bottom: 15px;
    color: #2196f3;
    font-size: 18px;
    border-bottom: 1px solid #2196f3;
    padding-bottom: 8px;
  }
  
  .monitor-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 15px;
  }
  
  .monitor-item {
    background-color: rgba(33, 150, 243, 0.2);
    padding: 15px;
    border-radius: 4px;
    text-align: center;
  }
  
  .monitor-label {
    display: block;
    font-size: 14px;
    opacity: 0.8;
    margin-bottom: 8px;
  }
  
  .monitor-value {
    display: block;
    font-size: 24px;
    font-weight: bold;
    color: #2196f3;
  }
  
  /* 视频监控面板样式 */
  .video-panel {
    position: absolute;
    right: 20px;
    width: 350px;
    color: #fff;
    padding: 15px;
    z-index: 1000;
  }
  
  #video-panel-1 {
    top: 60px;
  }
  
  #video-panel-2 {
    top: 380px;
  }
  
  .video-panel h3 {
    margin-top: 0;
    margin-bottom: 15px;
    color: #2196f3;
    font-size: 18px;
    border-bottom: 1px solid #2196f3;
    padding-bottom: 8px;
  }
  
  .video-placeholder {
    width: 100%;
    height: 150px;
    background-color: rgba(76, 78, 175, 0.2);
    border: 1px dashed #2196f3;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 16px;
    color: #888;
  }
  </style>
  