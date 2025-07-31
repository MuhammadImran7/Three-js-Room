<template>
    <div ref="threeContainer" class="three-container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

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
            const renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            this.$refs.threeContainer.appendChild(renderer.domElement);
            scene.background = new THREE.Color(0xeeeeee);

            // Floor
            const floorGeometry = new THREE.PlaneGeometry(200, 200);
            const floorMaterial = new THREE.MeshStandardMaterial({
                color: 0xffffff,
                side: THREE.DoubleSide,
                roughness: 0.5,
                metalness: 0.2
            });
            const floor = new THREE.Mesh(floorGeometry, floorMaterial);
            floor.rotation.x = -Math.PI / 2;
            floor.position.set(0, 0, 0);
            scene.add(floor);

            // Walls
            const wallMaterialSettings = {
                side: THREE.DoubleSide,
                roughness: 0.7,
                metalness: 0.1
            };
            
            const wall1 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0x00ff00, ...wallMaterialSettings })
            );
            wall1.position.set(0, 50, -100);
            scene.add(wall1);

            const wall2 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xff0000, ...wallMaterialSettings })
            );
            wall2.rotation.y = Math.PI / 2;
            wall2.position.set(100, 50, 0);
            scene.add(wall2);

            const wall3 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0x00ff00, ...wallMaterialSettings })
            );
            wall3.rotation.y = Math.PI / 2;
            wall3.position.set(-100, 50, 0);
            scene.add(wall3);

            const wall4 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xff0000, ...wallMaterialSettings })
            );
            wall4.position.set(0, 50, 100);
            scene.add(wall4);

            const walls = [
                { mesh: wall1, normal: new THREE.Vector3(0, 0, 1) },     // Back wall 
                { mesh: wall2, normal: new THREE.Vector3(-1, 0, 0) },    // Right wall
                { mesh: wall3, normal: new THREE.Vector3(1, 0, 0) },     // Left wall
                { mesh: wall4, normal: new THREE.Vector3(0, 0, -1) }     // Front wall
            ];

            // Lighting
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            const fillLight = new THREE.DirectionalLight(0xffffff, 1);
            fillLight.position.set(0, 100, 0);
            scene.add(fillLight);

            // Camera
            camera.position.set(0, 100, 200);
            camera.lookAt(0, 50, 0);

            // Controls
            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI/2;
            controls.minDistance = 260;
            controls.maxDistance = 500;

            // Model Loading with automatic scaling
            const gltfLoader = new GLTFLoader();
            
            const loadModel = (path, position, targetSize, rotation, callback) => {
                gltfLoader.load(path, (gltf) => {
                    const model = gltf.scene;
                    
                    // Calculate bounding box to get actual model size
                    const box = new THREE.Box3().setFromObject(model);
                    const currentSize = box.getSize(new THREE.Vector3());
                    
                    // Calculate scale factor based on target size
                    const scaleX = targetSize.x / currentSize.x;
                    const scaleY = targetSize.y / currentSize.y;
                    const scaleZ = targetSize.z / currentSize.z;
                    
                    // Use uniform scaling (smallest scale to maintain proportions)
                    const uniformScale = Math.min(scaleX, scaleY, scaleZ);
                    
                    model.scale.set(uniformScale, uniformScale, uniformScale);
                    model.position.set(position.x, position.y, position.z);
                    model.rotation.set(rotation.x, rotation.y, rotation.z);
                    
                    // Center the model at its position
                    const scaledBox = new THREE.Box3().setFromObject(model);
                    const center = scaledBox.getCenter(new THREE.Vector3());
                    model.position.sub(center);
                    model.position.add(new THREE.Vector3(position.x, position.y, position.z));
                    
                    model.traverse((child) => {
                        if (child.isMesh) {
                            child.castShadow = true;
                            child.receiveShadow = true;
                        }
                    });
                    
                    scene.add(model);
                    
                    if (callback) callback(model);
                    
                    console.log(`Loaded ${path}:`, 
                        `Original size: ${currentSize.x.toFixed(2)} x ${currentSize.y.toFixed(2)} x ${currentSize.z.toFixed(2)}, 
                         Scale factor: ${uniformScale.toFixed(2)}`);
                         
                }, undefined, (error) => {
                    console.error('Error loading model:', error);
                });
            };

            // Load models with proper target sizes (in scene units)
            // Bathtub - should be around 150x70x50 units
            loadModel('models/bath.glb', 
                { x: 50, y: 25, z: -70 }, 
                { x: 150, y: 50, z: 70 }, 
                { x: 0, y: 0, z: 0 }
            );

            // Door - should be around 80x200x5 units  
            loadModel('models/door.glb', 
                { x: 80, y: 37, z: 98 }, 
                { x: 5, y: 200, z: 100 }, 
                { x: 0, y: Math.PI / 2, z: 0 }
            );

            // Mirror - should be around 60x80x5 units
            loadModel('models/mirror.glb', 
                { x: -100, y: 60, z: 0 }, 
                { x: -60, y: 80, z: 45 }, 
                { x: 0, y: Math.PI/2, z: 0 }
            );

            // // Radiator - should be around 100x60x15 units
            // loadModel('models/radiator.glb', 
            //     { x: 0, y: 15, z: 95 }, 
            //     { x: 100, y: 60, z: 15 }, 
            //     { x: 0, y: 0, z: 0 }
            // );

            // // Shower - should be around 80x200x80 units
            // loadModel('models/shower.glb', 
            //     { x: 70, y: 0, z: -70 }, 
            //     { x: 80, y: 200, z: 80 }, 
            //     { x: 0, y: Math.PI / 4, z: 0 }
            // );

            // // Sink - should be around 60x40x50 units
            // loadModel('models/sink.glb', 
            //     { x: -70, y: 40, z: 50 }, 
            //     { x: 60, y: 40, z: 50 }, 
            //     { x: 0, y: Math.PI / 2, z: 0 }
            // );

            // // Toilet - should be around 40x70x60 units
            // loadModel('models/toilet.glb', 
            //     { x: 50, y: 0, z: 50 }, 
            //     { x: 40, y: 70, z: 60 }, 
            //     { x: 0, y: Math.PI / 2, z: 0 }
            // );

            // Helper function to create debug wireframes (optional)
            // const createDebugWireframe = (position, size, color = 0xff0000) => {
            //     const geometry = new THREE.BoxGeometry(size.x, size.y, size.z);
            //     const edges = new THREE.EdgesGeometry(geometry);
            //     const line = new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color }));
            //     line.position.set(position.x, position.y, position.z);
            //     // scene.add(line); // Uncomment to show debug wireframes
            // };

            // Add debug wireframes for reference (uncomment to see)
            // createDebugWireframe({ x: 0, y: 25, z: -50 }, { x: 150, y: 50, z: 70 }, 0x00ff00); // Bath
            // createDebugWireframe({ x: 100, y: 100, z: 0 }, { x: 5, y: 200, z: 80 }, 0xff0000); // Door

            // Animation
            const animate = () => {
                requestAnimationFrame(animate);
                const cameraPosition = new THREE.Vector3();
                camera.getWorldPosition(cameraPosition);

                const cameraDirection = new THREE.Vector3();
                camera.getWorldDirection(cameraDirection);

                walls.forEach(({ mesh, normal }) => {
                    const wallPosition = new THREE.Vector3();
                    mesh.getWorldPosition(wallPosition);

                    // Direction from camera to wall
                    const cameraToWall = new THREE.Vector3()
                        .subVectors(wallPosition, cameraPosition)
                        .normalize();

                    // If positive, wall is facing towards camera
                    const dot = cameraToWall.dot(normal);

                    mesh.visible = dot <= 0.1; // Small threshold for better control
                });
                
                fillLight.position.copy(camera.position);
                fillLight.position.y += 50; // Keep light above camera
                controls.update();
                renderer.render(scene, camera);
            };

            animate();

            // Window resize handler
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });
        }
    }
}
</script>

<style scoped>
.three-container {
    padding: 0;
    margin: 0;
}
</style>