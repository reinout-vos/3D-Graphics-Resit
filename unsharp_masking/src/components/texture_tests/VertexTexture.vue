<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';

const container = ref(null);

// Computes the light intensity for every vertex,
// store the result in a texture,
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

    // Light
    const light = new THREE.DirectionalLight(0xffffff, 1.0);
    light.position.set(5, 5, 5);
    scene.add(light);

    // Cube (don't add to scene)
    const cubeGeo = new THREE.BoxGeometry(2, 2, 2);
    const cubeMat = new THREE.MeshStandardMaterial({ color: 0x8888ff });
    const cube = new THREE.Mesh(cubeGeo, cubeMat);

    // Compute light intensity at each vertex
    const vertexNormals = cube.geometry.attributes.normal.array;
    const vertexPositions = cube.geometry.attributes.position.array;
    const lightIntensity = new Float32Array(cube.geometry.attributes.position.count);

    for (let i = 0, len = vertexPositions.length; i < len; i += 3) {
        const normal = new THREE.Vector3(vertexNormals[i], vertexNormals[i + 1], vertexNormals[i + 2]);
        const position = new THREE.Vector3(vertexPositions[i], vertexPositions[i + 1], vertexPositions[i + 2]);
        const lightDirection = light.position.clone().sub(position).normalize();
        const intensity = Math.max(lightDirection.dot(normal), 0);
        lightIntensity[i / 3] = intensity;
    }

    // Create data texture from light intensity
    const data = new Uint8Array(lightIntensity.length * 4);
    for (let i = 0; i < lightIntensity.length; i++) {
        const value = lightIntensity[i] * 255;
        data[i * 4] = value;     // R
        data[i * 4 + 1] = value; // G
        data[i * 4 + 2] = value; // B
        data[i * 4 + 3] = 255;   // A
    }

    var dataTex = new THREE.DataTexture(data, Math.sqrt(lightIntensity.length), Math.sqrt(lightIntensity.length), THREE.RGBAFormat);
    dataTex.magFilter = THREE.NearestFilter;
    dataTex.minFilter = THREE.NearestFilter;
    dataTex.needsUpdate = true;

    // Plane
    const planeGeo = new THREE.PlaneGeometry(15, 15);
    const planeMat = new THREE.MeshBasicMaterial({ map: dataTex, transparent: true });
    const plane = new THREE.Mesh(planeGeo, planeMat);
    scene.add(plane);

    // Render
    function animate() {
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
    }

    animate();
});
</script>

<style>
.container {
    width: 750px;
    height: 750px;
}
</style>
