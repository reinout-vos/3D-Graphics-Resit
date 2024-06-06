<script setup>
import RegularScene from './components/RegularScene.vue';
// import UnsharpMasking from './components/UnsharpMasking.vue';

import Test from './components/Test.vue';
</script>

<template>
  <div class="container">
    <RegularScene class="half-width"/>
    <Test class="half-width"/>
    <!-- <UnsharpMasking class="half-width"/> -->
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
