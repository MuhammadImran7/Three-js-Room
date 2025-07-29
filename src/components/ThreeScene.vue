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
            // Basic setup            
            const scene = new THREE.Scene();
            const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            const renderer = new THREE.WebGLRenderer();
            renderer.setSize(window.innerWidth, window.innerHeight);
            this.$refs.threeContainer.appendChild(renderer.domElement);
            // console.log(camera)
            // console.log(scene)
            // console.log(renderer)
            // Background color             
            scene.background = new THREE.Color(0xeeeeee);

            // Floor
            const floorGeometry = new THREE.PlaneGeometry(200, 200);
            // console.log(floorGeometry)
            const floorMaterial = new THREE.MeshStandardMaterial({
                color: 0xffffff,
                side: THREE.DoubleSide
            });
            // console.log(floorMaterial)
            const floor = new THREE.Mesh(floorGeometry, floorMaterial);
            floor.rotation.x = -Math.PI / 2;
            floor.position.set(0, 0, 0);
            scene.add(floor);
            // console.log(floor.position.x, floor.position.y, floor.position.z )
            // Wall materials
            const wallMaterial = new THREE.MeshLambertMaterial({
                color: 0x00FF00,
                side: THREE.DoubleSide
            });
            const wallMaterial1 = new THREE.MeshLambertMaterial({
                color: 0x0000FF,
                side: THREE.DoubleSide
            });
            const wallMaterial2 = new THREE.MeshLambertMaterial({
                color: 0xFF0000,
                side: THREE.DoubleSide
            });
            const wallMaterial3 = new THREE.MeshLambertMaterial({
                color: 0xFFFF00,
                side: THREE.DoubleSide
            });
            // console.log(wallMaterial, wallMaterial, wallMaterial2, wallMaterial3)
            // Create walls
            const wall1 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial);
            wall1.position.set(0, 50, -100); // Back wall
            scene.add(wall1);

            const wall2 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial1);
            wall2.rotation.y = Math.PI / 2;
            wall2.position.set(100, 50, 0); // Right wall
            scene.add(wall2);
            // console.log(wall2)
            const wall3 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial2);
            wall3.rotation.y = Math.PI / 2;
            wall3.position.set(-100, 50, 0); // Left wall
            scene.add(wall3);

            const wall4 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 4), wallMaterial3);
            wall4.position.set(0, 50, 100); // Front wall
            scene.add(wall4);

            // console.log(wall1, wall2, wall3, wall4)
            // FIXED: Correct wall normals (pointing INWARD to room)

            const walls = [
                { mesh: wall1, normal: new THREE.Vector3(0, 0, 1) },     // Back wall normal points forward
                { mesh: wall2, normal: new THREE.Vector3(-1, 0, 0) },    // Right wall normal points left
                { mesh: wall3, normal: new THREE.Vector3(1, 0, 0) },     // Left wall normal points right
                { mesh: wall4, normal: new THREE.Vector3(0, 0, -1) }     // Front wall normal points backward
            ];

            // console.log("Walls setup:", walls);

            // Lights           
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(50, 100, 50);
            scene.add(directionalLight);

            // Camera position (start INSIDE the room)
            camera.position.set(0, 0, 0); // Inside room center
            camera.lookAt(new THREE.Vector3(0, 30, -50)); // Look towards back wall

            // Mouse controls
            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2;
            controls.minDistance = 300;  // Zoom in limit (room ke andar nahi jayega)
            controls.maxDistance = 500; // Zoom out limit (invisible nahi hoga)

            //animation applied here
            const animate = () => {
                requestAnimationFrame(animate);
                const cameraPosition = new THREE.Vector3();

                camera.getWorldPosition(cameraPosition);

                const cameraDirection = new THREE.Vector3();
                camera.getWorldDirection(cameraDirection);

                camera.getWorldPosition(cameraPosition);

                walls.forEach(({ mesh, normal }) => {
                    const wallPosition = new THREE.Vector3();
                    mesh.getWorldPosition(wallPosition);

                    // Direction from camera to wall
                    const cameraToWall = new THREE.Vector3()
                        .subVectors(wallPosition, cameraPosition)
                        .normalize();

                    // agr positive ha to wall camer aki trf ha
                    const dot = cameraToWall.dot(normal);
                    // console.log(dot)
                    // Hide wall if it's facing the camera (blocking the view)
                    // Show wall if camera is behind it or at angle
                    mesh.visible = dot <= 0.1; // Small threshold for better control
                });
                directionalLight.position.set(cameraPosition.x, cameraPosition.y, cameraPosition.z);
                controls.update();
                renderer.render(scene, camera);
            };

            animate();
        }
    }
}
</script>

<style scoped>
.three-container {
    /* width: 100%; */
    /* height: 90vh; */
}
</style>