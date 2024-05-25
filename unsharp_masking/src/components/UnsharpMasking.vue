<template>
    <div ref="container" class="three-container"></div>
  </template>
  
  <script setup>
  import * as THREE from 'three';
  import { onMounted, ref } from 'vue';
  
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
  import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
  
  import { RGBShiftShader } from '../shaders/examples/RGBShiftShader.js'; // Adjust the path as needed
  
  const container = ref(null);
  
  let camera, renderer, composer;
  let object;
  let customBlurPass;
  

  // WORK IN PROGRESS! -> Does not work properly
  onMounted(() => {
      init();
      animate();
      window.addEventListener('resize', onWindowResize);
  })
  
  function init() {
      const width = container.value.clientWidth;
      const height = container.value.clientHeight;
  
      // Renderer
      renderer = new THREE.WebGLRenderer();
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(width, height);
      container.value.appendChild(renderer.domElement);
  
      // Camera
      camera = new THREE.PerspectiveCamera(70, width / height, 1, 1000);
      camera.position.z = 400;
  
      // Scene
      const scene = new THREE.Scene();
  
      // Object collection
      object = new THREE.Object3D();
      scene.add(object);
  
      // Load random objects
      const geometry = new THREE.BoxGeometry(1, 1, 1);
      const material = new THREE.MeshPhongMaterial({ color: 0xffffff, flatShading: true });
  
      for (let i = 0; i < 1; i++) {
          const mesh = new THREE.Mesh(geometry, material);
          mesh.position.set(Math.random() - 0.5, Math.random() - 0.5, Math.random() - 0.5).normalize();
          mesh.position.multiplyScalar(Math.random() * 400);
          mesh.rotation.set(Math.random() * 2, Math.random() * 2, Math.random() * 2);
          mesh.scale.x = mesh.scale.y = mesh.scale.z = 150;
          object.add(mesh);
      }
  
      // Light
      scene.add(new THREE.AmbientLight(0xcccccc));
  
      const light = new THREE.DirectionalLight(0xffffff, 3);
      light.position.set(1, 1, 1);
      scene.add(light);
  
      // Postprocessing
      composer = new EffectComposer(renderer);
      composer.addPass(new RenderPass(scene, camera));
  
      // TODO: create actual blurring shader
      customBlurPass = new ShaderPass(RGBShiftShader);
      customBlurPass.uniforms['amount'].value = 0.0015;
      composer.addPass(customBlurPass);
  
      const effect3 = new OutputPass();
      composer.addPass(effect3);
  }
  
  function onWindowResize() {
      const width = container.value.clientWidth;
      const height = container.value.clientHeight;
  
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
  
      renderer.setSize(width, height);
      composer.setSize(width, height);
      customBlurPass.uniforms['textureSize'].value.set(width, height);
  }
  
  function animate() {
      requestAnimationFrame(animate);
  
      object.rotation.x += 0.005;
      object.rotation.y += 0.01;
  
      composer.render();
  }
  
  function computeValencesAndRingIndices(geometry) {
      const vertices = geometry.attributes.position.array;
      const indices = geometry.index.array;
  
      const numVertices = vertices.length / 3;
      const valences = new Array(numVertices).fill(0);
      const ringIndices = new Array(numVertices).fill(null).map(() => []);
  
      for (let i = 0; i < indices.length; i += 3) {
          const a = indices[i];
          const b = indices[i + 1];
          const c = indices[i + 2];
  
          valences[a]++;
          valences[b]++;
          valences[c]++;
  
          ringIndices[a].push(b, c);
          ringIndices[b].push(a, c);
          ringIndices[c].push(a, b);
      }
  
      return { valences, ringIndices };
  }
  
  function createValenceSumTexture(geometry) {
      const { valences } = computeValencesAndRingIndices(geometry);
  
      const valenceSum = new Float32Array(valences.length);
      let cumulativeSum = 0;
      for (let i = 0; i < valences.length; i++) {
          cumulativeSum += valences[i];
          valenceSum[i] = cumulativeSum;
      }
  
      const texture = new THREE.DataTexture(valenceSum, valences.length, 1, THREE.RedFormat, THREE.FloatType);
      texture.needsUpdate = true;
  
      return texture;
  }
  
  function createRingIndicesTexture(geometry) {
      const { ringIndices } = computeValencesAndRingIndices(geometry);
  
      // Flatten the ringIndices array
      const flattenedRingIndices = [];
      for (let i = 0; i < ringIndices.length; i++) {
          flattenedRingIndices.push(...ringIndices[i]);
      }
  
      const texture = new THREE.DataTexture(new Float32Array(flattenedRingIndices), flattenedRingIndices.length, 1, THREE.RedFormat, THREE.FloatType);
      texture.needsUpdate = true;
  
      return texture;
  }
  </script>
  
  <style>
  .three-container {
    width: 100vw;
    height: 100vh;
  }
  </style>
  