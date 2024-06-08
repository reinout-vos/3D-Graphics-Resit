<template>
    <!-- <button @click="increaseIterations">Increase iterations</button> -->
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GUI } from 'dat.gui'

// Scene container
const container = ref(null);

// Mesh
let mesh, shaderMaterial;
const meshPath = "meshes/dragon_plane.gltf"


function calculateVertexIntensities(geometry, lightPosition) {
    geometry.computeVertexNormals();
    const positions = geometry.attributes.position.array;
    const normals = geometry.attributes.normal.array;
    const intensities = new Float32Array(positions.length / 3);

    for (let i = 0; i < positions.length; i += 3) {
        const vertex = new THREE.Vector3(positions[i], positions[i + 1], positions[i + 2]);
        const normal = new THREE.Vector3(normals[i], normals[i + 1], normals[i + 2]).normalize();
        const lightDir = new THREE.Vector3().subVectors(lightPosition, vertex).normalize();
        const intensity = Math.max(lightDir.dot(normal), 0);
        intensities[i / 3] = intensity;
    }

    return intensities;
}

function buildAdjacencyList(geometry) {
    const positions = geometry.attributes.position.array;
    const adjacencyList = Array.from({ length: positions.length / 3 }, () => []);

    const indexArray = geometry.index.array;
    for (let i = 0; i < indexArray.length; i += 3) {
        const a = indexArray[i];
        const b = indexArray[i + 1];
        const c = indexArray[i + 2];
        adjacencyList[a].push(b, c);
        adjacencyList[b].push(a, c);
        adjacencyList[c].push(a, b);
    }

    return adjacencyList.map(neighbors => [...new Set(neighbors)]);
}

function laplacianSmooth(intensities, adjacencyList, iterations = 1) {
    const smoothedIntensities = intensities.slice();

    for (let iter = 0; iter < iterations; iter++) {
        const newIntensities = smoothedIntensities.slice();

        for (let i = 0; i < intensities.length; i++) {
            const neighbors = adjacencyList[i];
            let sum = 0;
            for (let j = 0; j < neighbors.length; j++) {
                sum += smoothedIntensities[neighbors[j]];
            }
            newIntensities[i] = sum / neighbors.length;
        }

        for (let i = 0; i < smoothedIntensities.length; i++) {
            smoothedIntensities[i] = newIntensities[i];
        }
    }

    return smoothedIntensities;
}


onMounted(() => {
    const w = container.value.offsetHeight;
    const h = container.value.offsetHeight;

    // Scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x8f8181);

    // Camera
    const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 10000);
    camera.position.set(0.06, -1.6, -0.8);

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    renderer.shadowMap.enabled = true;
    container.value.appendChild(renderer.domElement);

    const controls = new OrbitControls( camera, renderer.domElement );

    // Light
    const light = new THREE.DirectionalLight(0xffffff, 1);
    light.position.set(4.5, 1.5, -0.1);
    light.castShadow = true;
    scene.add(light);

    // Point Light Helper
    const sphereSize = 1;
    const color = new THREE.Color(0xff0000);
    const pointLightHelper = new THREE.DirectionalLightHelper(light, sphereSize, color);
    scene.add(pointLightHelper);
    
    const manager = new THREE.LoadingManager();

    const loader = new GLTFLoader(manager);
    loader.load(meshPath, function (gltf) {
        const meshScene = gltf.scene;
        mesh = meshScene.children[0];
        mesh.castShadow = true
        mesh.position.set(0, 0, 0);
        mesh.scale.set(2,2,2);

        const plane = meshScene.children[1];
        plane.receiveShadow = true

        scene.add(plane)
        scene.add(mesh);

        // Set up GUI
        const params = {
            lambda: 1.0,
            sigma: 10,
            displaySmoothed: false
        };

        function updateLambda() {
            shaderMaterial.uniforms.lambda.value = params.lambda;
        }

        function updateSigma() {
            const smoothedIntensities = laplacianSmooth(intensities, adjacencyList, params.sigma)
            mesh.geometry.setAttribute('smoothed', new THREE.Float32BufferAttribute(smoothedIntensities, 1));
            shaderMaterial.needsUpdate = true
        }

        function updateDisplay(value) {
            if (value) {
                applySmoothed()
            } else {
                apply3DUnsharpMasking()
            }
        }

        const gui = new GUI()
        const unsharpFolder = gui.addFolder('3D Unsharp Masking')
        unsharpFolder.open();
        unsharpFolder.add(params, 'lambda', 0.0, 3.0).name('Lambda').onChange(updateLambda);
        unsharpFolder.add(params, 'sigma', 0, 50).step(5).name('Sigma').onChange(updateSigma);
        unsharpFolder.add(params, 'displaySmoothed').onChange(value => updateDisplay(value))

        const intensities = calculateVertexIntensities(mesh.geometry, light.position.clone())
        const adjacencyList = buildAdjacencyList(mesh.geometry)

        function applySmoothed() {
            const smoothedIntensities = laplacianSmooth(intensities, adjacencyList, params.sigma)

            mesh.geometry.setAttribute('smoothed', new THREE.Float32BufferAttribute(smoothedIntensities, 1));

            shaderMaterial = new THREE.ShaderMaterial({
                vertexShader: `
                    varying float vSmoothed;
                    attribute float smoothed;
                    
                    void main() {
                        vSmoothed = smoothed;
                        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                    }
                `,
                fragmentShader: `
                    varying float vSmoothed;

                    void main() {
                        gl_FragColor = vec4(vec3(vSmoothed), 1.0);
                    }
                `
            });

            mesh.material = shaderMaterial
        }

        function apply3DUnsharpMasking() {
            const smoothedIntensities = laplacianSmooth(intensities, adjacencyList, params.sigma)

            mesh.geometry.setAttribute('intensity', new THREE.Float32BufferAttribute(intensities, 1));
            mesh.geometry.setAttribute('smoothed', new THREE.Float32BufferAttribute(smoothedIntensities, 1));

            shaderMaterial = new THREE.ShaderMaterial({
                uniforms: {
                    lambda: { value: params.lambda }
                },
                vertexShader: `
                    varying float vIntensity;
                    attribute float intensity;

                    varying float vSmoothed;
                    attribute float smoothed;
                    
                    void main() {
                        vIntensity = intensity;
                        vSmoothed = smoothed;
                        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                    }
                `,
                fragmentShader: `
                    varying float vIntensity;
                    varying float vSmoothed;
                    uniform float lambda;

                    void main() {
                        float original = vIntensity;
                        float smoothed = vSmoothed;
                        vec3 mask = vec3(original + lambda * (original - smoothed));
                        gl_FragColor = vec4(mask, 1.0);
                    }
                `
            });

            mesh.material = shaderMaterial
        }

        if (params.displaySmoothed) {
            applySmoothed()
        } else {
            apply3DUnsharpMasking()
        }

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);

            controls.update()

            renderer.render(scene, camera);
        }

        animate();
    });
});
</script>
