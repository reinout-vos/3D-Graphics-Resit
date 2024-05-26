<script setup>
import RegularScene from './components/RegularScene.vue';
import UnsharpMasking from './components/UnsharpMasking.vue';
import VertexTexture from './components/texture_tests/VertexTexture.vue';
import ExampleTexture from './components/texture_tests/ExampleTexture.vue';

import PingPong from './components/texture_tests/PingPong.vue';
import PingPong2 from './components/texture_tests/PingPong2.vue';
</script>

<template>
  <!-- <div class="container">
    <RegularScene class="half-width"/>
    <UnsharpMasking class="half-width"/>
  </div> -->
  <div class="container">
    <!-- <VertexTexture /> -->
    <!-- <PingPong /> -->
    <PingPong2 />
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
