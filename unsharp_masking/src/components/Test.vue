<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js';
import { GUI } from 'dat.gui'
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

        // Set up GUI
        const params = {
            lambda: 1.0,
            sigma: 1.0,
        };

        const gui = new GUI()
        const unsharpFolder = gui.addFolder('3D Unsharp Masking')
        unsharpFolder.open();
        unsharpFolder.add(params, 'lambda', 0.0, 5.0).name('Lambda').onChange(updateUniforms);
        unsharpFolder.add(params, 'sigma', 0, 20).step(1).name('Sigma').onChange(updateUniforms);

        
        // ----- First pass: Render the mesh (now cube) to a texture with the vertex shader computing light intensity
        const firstPassScene = new THREE.Scene();
        firstPassScene.background = new THREE.Color(0x8f8181);
        
        const firstPassCamera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
        firstPassCamera.position.z = 5;

        const firstpassRt = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);

        const controls = new OrbitControls( firstPassCamera, renderer.domElement );

        
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
        firstPassScene.add(mesh);

        // ----- Second pass(es): Grab the light intensity texture and smooth
        // First texture stores at texel i the sum of all valences before vertex vi.
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

        if (!mesh.geometry.index) {
            mesh.geometry = BufferGeometryUtils.mergeVertices(mesh.geometry);
        }

        const neighbors = computeVertexNeighbors(mesh.geometry)

        const vertexCount = mesh.geometry.attributes.position.count
        
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


        // Define two render targets for pingponging
        const rt1 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);
        const rt2 = new THREE.WebGLRenderTarget(w, h, renderTargetOptions);

        let activeRt = rt1;
        let inactiveRt = rt2;

        // Laplacian smoothing shader
        const laplacianMaterial = new THREE.ShaderMaterial({
            uniforms: {
                tex: { value: null },
                valenceTexture: { value: valenceTexture },
                indicesTexture: { value: indicesTexture },
                textureWidth: { value: w}
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
                uniform float textureWidth;
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
                lambda: { value: params.lambda }
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
                uniform float lambda;
                varying vec2 vUv;

                void main() {
                    vec4 original = texture2D(originalTexture, vUv);
                    vec4 smoothed = texture2D(smoothedTexture, vUv);
                    vec4 unsharpMasked = original + lambda * (original - smoothed);

                    gl_FragColor = smoothed;
                }
            `,
        });

        // Render final scene to plane
        const planeGeometry = new THREE.PlaneGeometry(5, 5);
        const plane = new THREE.Mesh(planeGeometry, finalPassMaterial);
        scene.add(plane);

        function updateUniforms() {
            finalPassMaterial.uniforms.lambda.value = params.lambda;
            laplacianMaterial.needsUpdate = true;
        }

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);
            
            // First pass: create texture of light intensity values
            // firstPassScene -> firstpassRT
            renderer.setRenderTarget(firstpassRt);
            renderer.render(firstPassScene, firstPassCamera);

            // Second pass(es): smoothen the light intensity texture
            laplacianMaterial.uniforms.tex.value = firstpassRt.texture
            mesh.material = laplacianMaterial;
            
            // Render the scene to active render target
            renderer.setRenderTarget(activeRt);
            renderer.render(firstPassScene, firstPassCamera);

            // Swap active and inactive render targets
            let temp = activeRt;
            activeRt = inactiveRt;
            inactiveRt = temp;

            controls.update();

            // Final pass: compute unsharp masking and render to screen 
            // finalPassMaterial.uniforms.smoothedTexture.value = inactiveRt.texture;
            renderer.setRenderTarget(null);
            renderer.render(scene, camera);
        }

        animate();
    };
});
</script>
