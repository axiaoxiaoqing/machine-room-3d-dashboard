<template>
  <div id="app">
    <ThreeViewer />
  </div>
</template>

<script setup>
import ThreeViewer from './components/ThreeViewer.vue'
</script>

<style lang="less">
html,body,#app {
  height: 100%;
  width: 100%;
  padding: 0;
  margin: 0;
  overflow: hidden;
  -moz-user-select:none; /*火狐*/
  -webkit-user-select:none; /*webkit浏览器*/
  -ms-user-select:none; /*IE10*/
  -khtml-user-select:none; /*早期浏览器*/
  user-select:none;
}
*{
  -webkit-touch-callout:none;
  -webkit-user-select:none; 
  -moz-user-select:none;
  -ms-user-select:none;
  user-select:none;
  -webkit-tap-highlight-color:rgba(0,0,0,0);/**禁止div点击出现高亮底色*/
}
</style>
