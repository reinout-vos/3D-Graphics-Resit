<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';

const container = ref(null);

onMounted(() => {
    const w = 1024;
    const h = 1024;

    // Scene setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf1f1f1);

    // Camera setup
    const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
    camera.position.set(0, 0, 20);

    // Renderer setup
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    // Light setup
    const light = new THREE.DirectionalLight(0xffffff, 1.0);
    light.position.set(5, 5, 5);
    scene.add(light);

    const options = {
        minFilter: THREE.NearestFilter,
        magFilter: THREE.NearestFilter,
        format: THREE.RGBAFormat,
        type: THREE.FloatType
    };

    // Create two textures
    const textureA = new THREE.WebGLRenderTarget(w, h, options);
    const textureB = new THREE.WebGLRenderTarget(w, h, options);

    const material = new THREE.MeshBasicMaterial({ map: textureA.texture });
    const geometry = new THREE.PlaneGeometry(2, 2);
    const plane = new THREE.Mesh(geometry, material);
    scene.add(plane);

    let currentRenderTarget = textureA;
    let nextRenderTarget = textureB;

    let swapCount = 0;
    const maxSwaps = 20;

    function render() {
        if (swapCount >= maxSwaps) {
            // console.log("Completed 20 swaps.");
            return;
        }

        // Swap the render targets
        let temp = currentRenderTarget;
        currentRenderTarget = nextRenderTarget;
        nextRenderTarget = temp;

        // Render to the current render target
        renderer.setRenderTarget(currentRenderTarget);
        renderer.clear();
        renderer.render(scene, camera);

        // Display the updated texture on the plane
        plane.material.map = currentRenderTarget.texture;

        // Render to screen
        renderer.setRenderTarget(null);
        renderer.render(scene, camera);

        // Increment swap count and log
        swapCount++;
        // console.log(`Swap ${swapCount}: Current texture displayed.`);

        requestAnimationFrame(render);
    }

    render();


});
</script>
