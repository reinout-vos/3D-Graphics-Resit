<template>
    <div ref="container" class="container"></div>
</template>
  
<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';

const container = ref(null);

// Example: Creates a 1024x1024 texture of random values, 
// which is rendered to screen through a plane
onMounted(() => {
    const w = 1024;
    const h = 1024;
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
    camera.position.set(0, 0, 20);
  
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);
  
    // Data texture setup
    const side = 32;
    const amount = Math.pow(side, 2) * 4; // Adjust for RGBA values
    const data = new Uint8Array(amount);
    for (let i = 0; i < amount; i++) {
      data[i] = Math.random() * 256;
    }
  
    var dataTex = new THREE.DataTexture(data, side, side, THREE.RGBAFormat, THREE.UnsignedByteType);
    dataTex.magFilter = THREE.NearestFilter;
    dataTex.minFilter = THREE.NearestFilter; // Disable mipmapping
    dataTex.needsUpdate = true;
  
    // Plane setup
    const planeGeo = new THREE.PlaneGeometry(15, 15);
    const planeMat = new THREE.MeshBasicMaterial({ color: 0x4422ff, alphaMap: dataTex, transparent: true });
    const plane = new THREE.Mesh(planeGeo, planeMat);
    scene.add(plane);
  
    // Render
    renderer.render(scene, camera);
  });
</script>
  
<style>
.container {
    width: 750px;
    height: 750px;
}
</style>
  