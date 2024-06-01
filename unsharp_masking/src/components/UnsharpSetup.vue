<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

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

    let cube;

    const manager = new THREE.LoadingManager();

    const loader = new GLTFLoader(manager);
    loader.load('meshes/cube.gltf', function (gltf) {
        const scene = gltf.scene;
        cube = scene.children[0];
    });

    // Continue after the mesh is loaded
    manager.onLoad = function () {
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

        cube.material = shaderMaterial
        firstPassScene.add(cube);

        // ----- Second pass(es): Grab the light intensity texture and smooth
        // TODO: First texture stores at texel i the sum of all valences before vertex vi.
        // A second texture stores the 1-rings for all vertices as a concatenation of the individual indices.
        
        function computeVertexNeighbors(geometry) {
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

            return neighbors;
        }

        let geometry = cube.geometry

        const neighbors = computeVertexNeighbors(geometry)

        const vertexCount = geometry.attributes.position.count
        
        const valenceSummation = new Float32Array(vertexCount);
        const indices = [];

        let cumulativeValence = 0;
        for (let i = 0; i < vertexCount; i++) {
            const ni = neighbors[i].size;
            cumulativeValence += ni;
            valenceSummation[i] = cumulativeValence;

            // Add neighbors' indices to the indices array
            neighbors[i].forEach(index => indices.push(index));
        }

        // Convert indices to Float32Array for use in textures
        const indicesArray = new Float32Array(indices);

        // Create textures
        const valenceTexture = new THREE.DataTexture(valenceSummation, vertexCount, 1, THREE.RedFormat, THREE.FloatType);
        valenceTexture.needsUpdate = true;

        const indicesTexture = new THREE.DataTexture(indicesArray, indices.length, 1, THREE.RedFormat, THREE.FloatType);
        indicesTexture.needsUpdate = true;

        // TODO: Compute smoothed texture (now stored in firstpassRT.texture)


        // Define two render targets for pingponging
        const rt1 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);
        const rt2 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);

        let activeRt = rt1;
        let inactiveRt = rt2;

        const laplacianMaterial = new THREE.ShaderMaterial({
            uniforms: {
                tex: { value: null },
                valenceTexture: { value: valenceTexture },
                indicesTexture: { value: indicesTexture },
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
                uniform sampler2D valenceTexture;
                uniform sampler2D indicesTexture;
                varying vec2 vUv;

                void main() {
                    float valence = texture2D(valenceTexture, vUv).r;
                    float sumNeighbors = 0.0;

                    for (int i = 0; i < int(valence); i++) {
                        float neighborIndex = texture2D(indicesTexture, vec2(float(i) / float(textureWidth), vUv.y)).r;
                        sumNeighbors += texture2D(tex, vec2(neighborIndex / float(textureWidth), 0.5)).r;
                    }

                    float average = sumNeighbors / valence;
                    float originalIntensity = texture2D(tex, vUv).r;
                    float laplacian = originalIntensity - average;

                    gl_FragColor = vec4(laplacian, laplacian, laplacian, 1.0);
                }
            `
        });


        // ----- Final pass: Render the texture to a plane
        // TODO: Read the smoothed texture and compute unsharp masking 
        const finalPassMaterial = new THREE.ShaderMaterial({
            uniforms: {
                originalTexture: { value: firstpassRt.texture },
                smoothedTexture: { value: activeRt.texture },
            },
            vertexShader: `
                varying vec2 vUv;

                void main() {
                    vUv = uv;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform sampler2D originalTexture;
                uniform sampler2D smoothedTexture;
                varying vec2 vUv;

                void main() {
                    vec4 original = texture2D(originalTexture, vUv);
                    vec4 smoothed = texture2D(smoothedTexture, vUv);
                    float lambda = 1.0;
                    vec4 unsharpMasked = original + lambda * (original - smoothed);

                    gl_FragColor = unsharpMasked;
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
                laplacianMaterial.uniforms.tex.value = i === 0 ? firstpassRt.texture : activeRt.texture;
                laplacianMaterial.needsUpdate = true;

                // Render the scene to active render target
                renderer.setRenderTarget(activeRt);
                renderer.render(firstPassScene, firstPassCamera);

                // Update the material of the plane to use active render target texture
                // plane.material.uniforms.tex.value = activeRt.texture;
                
                // Swap active and inactive render targets
                let temp = activeRt;
                activeRt = inactiveRt;
                inactiveRt = temp;
            }

            // Final pass: compute unsharp masking and render to screen 
            finalPassMaterial.uniforms.smoothedTexture.value = activeRt.texture;
            renderer.setRenderTarget(null);
            renderer.render(scene, camera);
        }

        animate();
    };
});
</script>

<style>
.container {
    width: 1024px;
    height: 1024px;
}
</style>
