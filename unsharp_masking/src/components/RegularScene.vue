<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';


// Scene container
const container = ref(null);

// Mesh
let mesh;
const meshPath = "meshes/dragon2.gltf"

// Render target options
const renderTargetOptions = {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
    type: THREE.UnsignedByteType,
    stencilBuffer: false,
    depthBuffer: true
};

onMounted(() => {
    const w = container.value.offsetHeight;
    const h = container.value.offsetHeight;

    // Scene
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x8f8181);

    const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 10000);
    camera.position.z = 5;

    // Renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    const controls = new OrbitControls( camera, renderer.domElement );

    // Light
    const light = new THREE.PointLight(0xffffff, 1, 100);
    // light.position.set(1, -0.2, 1);
    light.position.set(0, 0, 1);
    scene.add(light);

    // Point Light Helper
    // const sphereSize = 1;
    // const color = new THREE.Color(0xff0000);
    // const pointLightHelper = new THREE.PointLightHelper(light, sphereSize, color);
    // scene.add(pointLightHelper);
    
    const manager = new THREE.LoadingManager();

    const loader = new GLTFLoader(manager);
    loader.load(meshPath, function (gltf) {
        const scene = gltf.scene;
        mesh = scene.children[0];
    });

    // Continue after the mesh is loaded
    manager.onLoad = function () {
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

        mesh.material = shaderMaterial
        scene.add(mesh);

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);
            
            if (mesh) {
                // mesh.rotation.z += 0.001;
            }

            renderer.setRenderTarget(null);
            renderer.render(scene, camera);
        }

        animate();
    };
});
</script>
