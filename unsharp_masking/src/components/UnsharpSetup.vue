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
    scene.background = new THREE.Color(0x8f8181);

    const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 10000);
    camera.position.z = 5;

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    // Light
    const light = new THREE.PointLight(0xffffff, 1, 100);
    light.position.set(1, 0, 1);
    scene.add(light);

    // Point Light Helper
    const sphereSize = 1;
    const color = new THREE.Color(0xff0000);
    const pointLightHelper = new THREE.PointLightHelper(light, sphereSize, color);
    scene.add(pointLightHelper);


    // ----- First pass: Render the cube to a texture with the vertex shader computing light intensity
    const firstPassScene = new THREE.Scene();
    firstPassScene.background = new THREE.Color(0x8f8181);
    const firstPassCamera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
    firstPassCamera.position.z = 5;

    const cubeGeometry = new THREE.BoxGeometry();
    const texturedMaterial = new THREE.ShaderMaterial({
        uniforms: {
            lightDirection: { value: light.position.clone().normalize() },
        },
        vertexShader: `
            varying float vLightIntensity;

            uniform vec3 lightDirection;

            void main() {
                vec3 transformedNormal = normalize((modelViewMatrix * vec4(normal, 0.0)).xyz);
                vLightIntensity = max(dot(transformedNormal, lightDirection), 0.0);
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `,
        fragmentShader: `
            varying float vLightIntensity;

            void main() {
                gl_FragColor = vec4(vec3(vLightIntensity), 1.0);
            }
        `,
    });

    const cube = new THREE.Mesh(cubeGeometry, texturedMaterial);
    firstPassScene.add(cube);

    // Render target options
    const renderTargetOptions = {
        minFilter: THREE.LinearFilter,
        magFilter: THREE.LinearFilter,
        format: THREE.RGBAFormat,
        type: THREE.UnsignedByteType,
        stencilBuffer: false,
        depthBuffer: false
    };

    const firstpassRt = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);



    // ----- TODO: Compute smoothed texture (now stored in rt1.texture)
    // Define two render targets for pingponging
    const rt1 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);
    const rt2 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);

    let activeRt = rt1;
    let inactiveRt = rt2;


    // ----- Final pass: Render the texture to a plane
    const secondPassMaterial = new THREE.ShaderMaterial({
        uniforms: {
            tex: { value: activeRt.texture },
        },
        vertexShader: `
            varying vec2 vUv;

            void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `,
        fragmentShader: `
            varying vec2 vUv;
            uniform sampler2D tex;

            void main() {
                vec4 texel = texture2D(tex, vUv);
                gl_FragColor = vec4(texel.rgb, 1.0);
            }
        `,
    });

    // Render final scene to plane
    const planeGeometry = new THREE.PlaneGeometry(5, 5);
    const plane = new THREE.Mesh(planeGeometry, secondPassMaterial);
    scene.add(plane);

    function animate() {
        requestAnimationFrame(animate);
        cube.rotation.x += 0.01;
        cube.rotation.y += 0.01;

        // First pass: create texture of light intensity values
        renderer.setRenderTarget(firstpassRt);
        renderer.render(firstPassScene, firstPassCamera);

        // Second pass(es): smoothen the light intensity texture
        for (let i = 0; i < 10; i++) {
            // Render the scene to active render target
            renderer.setRenderTarget(activeRt);
            renderer.render(firstPassScene, firstPassCamera);

            // Update the material of the plane to use active render target texture
            plane.material.uniforms.tex.value = activeRt.texture;
            plane.material.needsUpdate = true;
            
            // Swap active and inactive render targets
            let temp = activeRt;
            activeRt = inactiveRt;
            inactiveRt = temp;
        }

        // Final pass: compute unsharp masking and render to screen 
        renderer.setRenderTarget(null);
        renderer.render(scene, camera);
    }

    animate();
});
</script>

<style>
.container {
    width: 1024px;
    height: 1024px;
}
</style>
