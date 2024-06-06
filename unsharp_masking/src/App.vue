<script setup>
import RegularScene from './components/RegularScene.vue';
// import UnsharpMasking from './components/UnsharpMasking.vue';

import PreProcess from './components/PreProcess.vue'
</script>

<template>
  <!-- <div class="container">
    <RegularScene class="half-width"/>
    <UnsharpMasking class="half-width"/>
  </div> -->
  <PreProcess />
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
