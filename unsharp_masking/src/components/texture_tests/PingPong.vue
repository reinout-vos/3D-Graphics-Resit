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

    // Scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf1f1f1);
    
    // Camera
    const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
    camera.position.set(0, 0, 50);

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    // Light
    const light = new THREE.DirectionalLight(0xffffff, 1.0);
    light.position.set(5, 5, 5);
    scene.add(light);

    // --------------- Define texture of random values --------------- 
    const size = 32;
    const data = new Uint8Array(size * size * 4);
    for (let i = 0; i < data.length; i++) {
        data[i] = Math.random() * 255;
    }

    const initialTexture = new THREE.DataTexture(data, size, size, THREE.RGBAFormat);
    initialTexture.needsUpdate = true;

    // --------------- Define render targets ---------------
    const renderTargetOptions = {
        minFilter: THREE.NearestFilter,
        magFilter: THREE.NearestFilter,
        format: THREE.RGBAFormat,
        type: THREE.UnsignedByteType,
        stencilBuffer: false,
        depthBuffer: false
    };
    const rt1 = new THREE.WebGLRenderTarget(size, size, renderTargetOptions);
    const rt2 = new THREE.WebGLRenderTarget(size, size, renderTargetOptions);
    
    // --------------- Define Plane ---------------
    const planeGeometry = new THREE.PlaneGeometry(50, 50);
    const materialUsingInitialTexture = new THREE.ShaderMaterial({
        uniforms: {
            tex: { value: initialTexture }
        },
        vertexShader: `
            varying vec2 vUv;
            
            void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `,
        fragmentShader: `
            uniform sampler2D tex;
            varying vec2 vUv;
            
            void main() {
                vec4 texel = texture2D(tex, vUv);
                gl_FragColor = vec4(texel.rgb, 0.9);
            }
        `
    });

    const plane = new THREE.Mesh(planeGeometry, materialUsingInitialTexture);
    scene.add(plane);

    // --------------- PingPong ---------------
    // Render the scene to first render target
    renderer.setRenderTarget(rt1);
    renderer.render(scene, camera);

    // Update the material of the plane to use first render target texture
    plane.material.uniforms.tex.value = rt1.texture;
    plane.material.needsUpdate = true;

    // Render the scene to second render target
    renderer.setRenderTarget(rt2);
    renderer.render(scene, camera);

    // Update the material of the plane to use second render target texture
    plane.material.uniforms.tex.value = rt2.texture;
    plane.material.needsUpdate = true;

    // Reset the render target to null to render to the canvas
    renderer.setRenderTarget(null);
    renderer.render(scene, camera);
});

</script>

<style>
.container {
    width: 750px;
    height: 750px;
}
</style>
