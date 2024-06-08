<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Scene container
const container = ref(null);

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
        const mesh = meshScene.children[0];
        mesh.castShadow = true
        mesh.position.set(0, 0, 0);
        mesh.scale.set(2,2,2);

        const plane = meshScene.children[1];
        plane.receiveShadow = true

        scene.add(plane)
        scene.add(mesh);

        const intensities = calculateVertexIntensities(mesh.geometry, light.position.clone())
        mesh.geometry.setAttribute('intensity', new THREE.Float32BufferAttribute(intensities, 1));

        const shaderMaterial = new THREE.ShaderMaterial({
            vertexShader: `
                varying float vIntensity;
                attribute float intensity;

                void main() {
                    vIntensity = intensity;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                varying float vIntensity;

                void main() {
                    gl_FragColor = vec4(vec3(vIntensity), 1.0);
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
});
</script>
