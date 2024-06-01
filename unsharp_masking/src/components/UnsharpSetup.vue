<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import {OBJLoader} from 'three/addons/loaders/OBJLoader.js';
import {MTLLoader} from 'three/addons/loaders/MTLLoader.js';
import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js';
// Scene container
const container = ref(null);

// Render target options
const renderTargetOptions = {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
    type: THREE.UnsignedByteType,
    stencilBuffer: false,
    depthBuffer: false
};

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
    light.position.set(1, -0.2, 1);
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

    const firstpassRt = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);
    
    // Assign light intensity shader
    const shaderMaterial = new THREE.ShaderMaterial({
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
    
    let cube;

    const manager = new THREE.LoadingManager();
    const mtlLoader = new MTLLoader(manager);
    const objLoader = new OBJLoader(manager);

    mtlLoader.load('meshes/cube.mtl', (mtl) => {
        mtl.preload();
        objLoader.setMaterials(mtl);
        objLoader.load('meshes/cube.obj', (root) => {
            root.traverse((child) => {
                if (child.isMesh) {
                    child.material = shaderMaterial;
                    cube = child;
                }
            });
            firstPassScene.add(cube);
        });
    });

    // ----- Second pass(es): Grab the light intensity texture and smooth
    // TODO: First texture stores at texel i the sum of all valences before vertex vi.
    // A second texture stores the 1-rings for all vertices as a concatenation of the individual indices.
    
    function computeVertexNeighbors(geometry) {
        if (!geometry.index) {
            geometry = BufferGeometryUtils.mergeVertices(geometry);
        }

        const index = geometry.index.array;
        const position = geometry.attributes.position.array;
        const vertexCount = position.length / 3;
        const neighbors = Array(vertexCount).fill().map(() => new Set());

        // Iterate over each face (triangle)
        for (let i = 0; i < index.length; i += 3) {
            const a = index[i];
            const b = index[i + 1];
            const c = index[i + 2];

            // Add neighbors for vertex a
            neighbors[a].add(b);
            neighbors[a].add(c);

            // Add neighbors for vertex b
            neighbors[b].add(a);
            neighbors[b].add(c);

            // Add neighbors for vertex c
            neighbors[c].add(a);
            neighbors[c].add(b);
        }

        // Count neighbors for each vertex
        const neighborCounts = neighbors.map(neighborSet => neighborSet.size);

        return neighborCounts;
    }

    // Continue after the mesh is loaded
    manager.onLoad = function () {
        // TODO: correct this
        const valences = computeVertexNeighbors(cube.geometry);

        const valenceSum = new Float32Array(cube.geometry.attributes.position.count);
        valenceSum[0] = valences[0];
        for (let i = 1; i < valences.length; i++) {
            valenceSum[i] = valenceSum[i - 1] + valences[i];
        }

        const valenceSumTexture = new THREE.DataTexture(valenceSum, valences.length, 1, THREE.RedFormat, THREE.FloatType);
        valenceSumTexture.needsUpdate = true;
    };

    // TODO: Compute smoothed texture (now stored in firstpassRT.texture)


    // Define two render targets for pingponging
    const rt1 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);
    const rt2 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);

    let activeRt = rt1;
    let inactiveRt = rt2;


    // ----- Final pass: Render the texture to a plane
    // TODO: Read the smoothed texture and compute unsharp masking 
    const finalPassMaterial = new THREE.ShaderMaterial({
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
    const plane = new THREE.Mesh(planeGeometry, finalPassMaterial);
    scene.add(plane);

    function animate() {
        requestAnimationFrame(animate);
        
        if (cube) {
            cube.rotation.x += 0.01;
            cube.rotation.y += 0.01;
        }

        // First pass: create texture of light intensity values
        renderer.setRenderTarget(firstpassRt);
        renderer.render(firstPassScene, firstPassCamera);

        // Second pass(es): smoothen the light intensity texture
        for (let i = 0; i < 1; i++) {
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
