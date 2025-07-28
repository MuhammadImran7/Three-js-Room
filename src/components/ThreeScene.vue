<template>
    <div ref="threeContainer" class="three-container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';


export default {
    name: 'ThreeScene',
    mounted() {
        this.initThreeJS();
    },
    methods: {
        initThreeJS() {
            // lazmi lazmi chezain            
            const scene = new THREE.Scene();
            const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            const renderer = new THREE.WebGLRenderer();
            renderer.setSize(window.innerWidth, window.innerHeight);
            this.$refs.threeContainer.appendChild(renderer.domElement);

            // bg color             
            scene.background = new THREE.Color(0xeeeeee);

            // Floor settings            
            const floorGeometry = new THREE.PlaneGeometry(200, 200); //
            const floorMaterial = new THREE.MeshLambertMaterial({
                color: 0x707000,
                side: THREE.DoubleSide  // Both sides visible
            });
            const floor = new THREE.Mesh(floorGeometry, floorMaterial);

            // FIXED: Correct rotation direction
            floor.rotation.x = -Math.PI / 2;  // Negative lagaya 
            floor.position.set(0, 0, 0);
            scene.add(floor);

            //wall ka material
            const wallMaterial = new THREE.MeshLambertMaterial({
                color: 0x00FF00,
                side: THREE.DoubleSide
            });
            const wallMaterial1 = new THREE.MeshLambertMaterial({
                color: 0x0000FF,
                side: THREE.DoubleSide
            });
            const wallMaterial2 = new THREE.MeshLambertMaterial({
                color: 0x0000FF,
                side: THREE.DoubleSide
            });
            const wallMaterial3 = new THREE.MeshLambertMaterial({
                color: 0x00FF00,
                side: THREE.DoubleSide
            });

            const wall1 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial);
            wall1.position.set(0, 50, -100);
            scene.add(wall1);

            const wall2 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial1);
            wall2.rotation.y = Math.PI / 2;
            wall2.position.set(100, 50, 0);
            scene.add(wall2);

            const wall3 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial2);
            wall3.rotation.y = Math.PI / 2;
            wall3.position.set(-100, 50, 0);
            scene.add(wall3);

            const wall4 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial3);
            wall4.position.set(0, 50, 100);
            scene.add(wall4);
            // Lights           
            const ambientLight = new THREE.AmbientLight(0x404040, 1); // Reduced intensity
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(50, 100, 50); // Better position
            scene.add(directionalLight);

            // Camera ki position set ki            
            camera.position.set(300, 400, 150);
            camera.lookAt(new THREE.Vector3(0, 0, 0));

            // Mouse Control apply kya

            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;  // Smooth movement
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2;

            renderer.render(scene, camera);

            const animate = () => {
                requestAnimationFrame(animate);
                controls.update(); // OrbitControls update karne ke liye
                renderer.render(scene, camera);
            };
            animate();

        }
    }
} 
</script>

<style scoped>
.three-container {
    width: 100%;
    height: 100vh;
}
</style>