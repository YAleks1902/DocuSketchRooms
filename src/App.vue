<template>
  <div id="app">
    <CanvasRenderer ref="canvasRenderer"></CanvasRenderer>
    <ButtonComponent buttonText="Next Wall" @button-click="nextWall"></ButtonComponent>
    <ButtonComponent buttonText="Change Room" @button-click="changeRoom"></ButtonComponent>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import CanvasRenderer from './components/CanvasRenderer.vue'
import ButtonComponent from './components/ButtonComponent.vue'

const canvasRenderer = ref(null)

const nextWall = () => {
  canvasRenderer.value.nextWall()
}

const changeRoom = () => {
  canvasRenderer.value.changeRoom()
}
</script>

<style>
#app {
  text-align: center;
  margin-top: 20px;
}
</style>
