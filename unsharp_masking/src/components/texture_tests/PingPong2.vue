<template>
    <div ref="container" class="container"></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, ref } from 'vue';

const container = ref(null);

// From: https://codepen.io/forerunrun/pen/poONERw
onMounted(() => {
    // Dynamically use the container's dimensions
    const w = container.value.clientWidth;
    const h = container.value.clientHeight;

    // Scene setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf1f1f1);

    // Camera setup
    const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 10000);
    camera.position.set(0, 0, 20);

    // Renderer setup
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(w, h);
    container.value.appendChild(renderer.domElement);

    // Light setup
    const light = new THREE.DirectionalLight(0xffffff, 1.0);
    light.position.set(5, 5, 5);
    scene.add(light);

    
    var bufferScene;
    var textureA;
    var textureB;
    var bufferMaterial;
    var plane;
    var bufferObject;
    var finalMaterial;
    var quad;

    function buffer_texture_setup(){
        //Create buffer scene
        bufferScene = new THREE.Scene();
        //Create 2 buffer textures
        textureA = new THREE.WebGLRenderTarget( window.innerWidth, window.innerHeight, { minFilter: THREE.LinearFilter, magFilter: THREE.NearestFilter});
        textureB = new THREE.WebGLRenderTarget( window.innerWidth, window.innerHeight, { minFilter: THREE.LinearFilter, magFilter: THREE.NearestFilter} );
        //Pass textureA to shader
        bufferMaterial = new THREE.ShaderMaterial( {
            uniforms: {
                res : {type: 'v2',value:new THREE.Vector2(window.innerWidth,window.innerHeight)},//Keeps the resolution
                bufferTexture: { type: "t", value: textureA },
                smokeSource: {type:"v3",value:new THREE.Vector3(0,0,0)}///This keeps the position of the mouse and whether it was clicked or not
            },
            fragmentShader: `
                uniform vec2 res;
                uniform sampler2D bufferTexture;
                uniform vec3 smokeSource;
		
                void main() {
	                vec2 pixel = gl_FragCoord.xy / res.xy;
	                gl_FragColor = texture2D( bufferTexture, pixel );

        	        float dist = distance(smokeSource.xy,gl_FragCoord.xy);
        			gl_FragColor.rgb += smokeSource.z * max(15.0-dist,0.0);
  
        	        float xPixel = 1.0/res.x;//The size of a single pixel
	                float yPixel = 1.0/res.y;
                    vec4 rightColor = texture2D(bufferTexture,vec2(pixel.x+xPixel,pixel.y));
                    vec4 leftColor = texture2D(bufferTexture,vec2(pixel.x-xPixel,pixel.y));
                    vec4 upColor = texture2D(bufferTexture,vec2(pixel.x,pixel.y+yPixel));
                    vec4 downColor = texture2D(bufferTexture,vec2(pixel.x,pixel.y-yPixel));
        
                    float factor = 8.0 * 0.016 * (leftColor.r + rightColor.r + downColor.r * 3.0 + upColor.r - 6.0 * gl_FragColor.r);
		 
                    float minimum = 0.003;
                    if(factor >= -minimum && factor < 0.0) factor = -minimum;

                    gl_FragColor.rgb += factor;
                }
            `
        } );
        plane = new THREE.PlaneGeometry( window.innerWidth, window.innerHeight );
        bufferObject = new THREE.Mesh( plane, bufferMaterial );
        bufferScene.add(bufferObject);

        //Draw textureB to screen 
        finalMaterial =  new THREE.MeshBasicMaterial({map: textureB.texture});
        quad = new THREE.Mesh( plane, finalMaterial );
        scene.add(quad);
    }
    buffer_texture_setup();


    const updateMousePosition = (event) => {
        bufferMaterial.uniforms.smokeSource.value.x = event.clientX;
        bufferMaterial.uniforms.smokeSource.value.y = window.innerHeight - event.clientY;
    };

    const handleMouseDown = () => {
        bufferMaterial.uniforms.smokeSource.value.z = 0.1;
    };

    const handleMouseUp = () => {
        bufferMaterial.uniforms.smokeSource.value.z = 0;
    };

    // Attach event listeners directly on the container
    container.value.addEventListener('mousemove', updateMousePosition);
    container.value.addEventListener('mousedown', handleMouseDown);
    container.value.addEventListener('mouseup', handleMouseUp);

    // Animation loop and rendering
    const render = () => {
        requestAnimationFrame(render);

        //Draw to textureB
        renderer.setRenderTarget( textureB );
        renderer.render(bufferScene,camera);
        
        //Swap textureA and B 
        var t = textureA;
        textureA = textureB;
        textureB = t;
        quad.material.map = textureB.texture;
        bufferMaterial.uniforms.bufferTexture.value = textureA.texture;

        //Finally, draw to the screen
        renderer.setRenderTarget( null );
        renderer.render( scene, camera );

    };
    render();
});
</script>

<style>
.container {
    width: 100%;
    height: 100%;
}
</style>
