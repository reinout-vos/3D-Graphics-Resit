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
    // const size = 32;
    // const data = new Uint8Array(size * size * 4);
    // for (let i = 0; i < data.length; i++) {
    //     data[i] = Math.random() * 255;
    // }

    // const initialTexture = new THREE.DataTexture(data, size, size, THREE.RGBAFormat);
    // initialTexture.needsUpdate = true;

    // --------------- Define texture of cube light intensity values --------------- 
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

    const size = Math.sqrt(lightIntensity.length)

    var initialTexture = new THREE.DataTexture(data, size, size, THREE.RGBAFormat);
    initialTexture.magFilter = THREE.NearestFilter;
    initialTexture.minFilter = THREE.NearestFilter;
    initialTexture.needsUpdate = true;
    
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
                gl_FragColor = texel;
            }
        `
    });

    const plane = new THREE.Mesh(planeGeometry, materialUsingInitialTexture);
    scene.add(plane);

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
    
    // --------------- PingPong ---------------
    let activeRt = rt1;
    let inactiveRt = rt2;

    for (let i = 0; i < 0; i++) {
        // Render the scene to active render target
        renderer.setRenderTarget(activeRt);
        renderer.render(scene, camera);

        // Update the material of the plane to use active render target texture
        plane.material.uniforms.tex.value = activeRt.texture;
        plane.material.needsUpdate = true;
        
        // Swap active and inactive render targets
        let temp = activeRt;
        activeRt = inactiveRt;
        inactiveRt = temp;
    }

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
