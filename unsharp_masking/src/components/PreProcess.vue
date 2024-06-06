<template>
    <!-- <button @click="increaseIterations">Increase iterations</button> -->
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// import intensities from '../../preprocessing/dragon_intensities';
import adjacencyList from '../../preprocessing/dragon_adjacencylist';

// Scene container
const container = ref(null);

const iterations = ref(0);

// const increaseIterations = () => {
//     iterations.value += 10
//     console.log(laplacianSmooth(intensities, adjacencyList, iterations.value))
// }

// Mesh
let mesh;
const meshPath = "meshes/dragon2.gltf"

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

    // Axes
    const axesHelper = new THREE.AxesHelper( 5 );
    scene.add( axesHelper );

    // Camera
    const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 10000);
    // camera.position.set(0, 0, 0);
    camera.position.set(0.06, -1.6, -0.8);
    // Rotate the camera
    // camera.rotateX(-Math.PI / 4);

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    const controls = new OrbitControls( camera, renderer.domElement );

    // Light
    // const light = new THREE.DirectionalLight(0xffffff, 1);
    // light.position.set(1, 1, 1.5);
    // scene.add(light);

    const light = new THREE.PointLight(0xffffff, 1, 0);
    light.position.set(5, -0.6, -5.1);
    scene.add(light);

    // Point Light Helper
    const sphereSize = 1;
    const color = new THREE.Color(0xff0000);
    const pointLightHelper = new THREE.PointLightHelper(light, sphereSize, color);
    scene.add(pointLightHelper);
    
    const manager = new THREE.LoadingManager();

    const loader = new GLTFLoader(manager);
    loader.load(meshPath, function (gltf) {
        const meshScene = gltf.scene;
        mesh = meshScene.children[0];
        // Apply the mesh transformations
        

        mesh.position.set(0, 0, 0);
        // mesh.scale.set(2,2,2);
        // mesh.rotation.z = Math.PI / 2;
        // Rotate the mesh
        // mesh.rotateX(-Math.PI / 2);

        scene.add(mesh);

        const intensities = calculateVertexIntensities(mesh.geometry, light.position.clone())
        const smoothedIntensities = laplacianSmooth(intensities, adjacencyList, 10)

        mesh.geometry.setAttribute('intensity', new THREE.Float32BufferAttribute(intensities, 1));
        mesh.geometry.setAttribute('smoothed', new THREE.Float32BufferAttribute(smoothedIntensities, 1));

        const shaderMaterial = new THREE.ShaderMaterial({
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
                void main() {
                    float original = vIntensity;
                    float smoothed = vSmoothed;
                    float lambda = 0.5;
                    vec3 mask = vec3(original + lambda * (original - smoothed));
                    gl_FragColor = vec4(mask, 1.0);
                }
            `
        });

        mesh.material = shaderMaterial

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);

            controls.update()
            
            renderer.render(scene, camera);
        }

        animate();
    });

    // // Continue after the mesh is loaded
    // manager.onLoad = function () {

        
    // };
});
</script>
