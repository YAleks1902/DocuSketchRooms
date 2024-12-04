<template>
  <button @click="buttonClicked">{{ buttonText }}</button>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue'

defineProps({
  buttonText: String,
})

const emit = defineEmits(['button-click'])

const buttonClicked = () => {
  emit('button-click')
}
</script>

<style scoped>
button {
  margin: 10px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>
