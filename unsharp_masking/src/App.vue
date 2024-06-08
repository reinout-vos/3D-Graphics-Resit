<script setup>
import RegularScene from './components/RegularScene.vue';
import PreProcess from './components/PreProcess.vue'
</script>

<template>
  <div class="container">
    <RegularScene class="half-width"/>
    <PreProcess class="half-width"/>
  </div>
</template>

<style>
.container {
  display: flex;
  width: 100vw;
  height: 100vh;
}

.half-width {
  width: 50%;
  height: 100%;
}
</style>
