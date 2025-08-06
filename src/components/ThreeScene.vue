<template>
    <div ref="threeContainer" class="three-container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

export default {
    name: 'ThreeScene',
    data() {
        return {
            draggableObjects: [],
            isDragging: false,
            dragObject: null,
            mouse: new THREE.Vector2(),
            raycaster: new THREE.Raycaster(),
            dragPlane: new THREE.Plane(),
            dragOffset: new THREE.Vector3(),
            roomBounds: {
                minX: -95, maxX: 95,
                minZ: -95, maxZ: 95,
                minY: 50, maxY: 100
            }
        }
    },
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

            // Store references for drag functionality
            this.scene = scene;
            this.camera = camera;
            this.renderer = renderer;

            // Floor
            const floorGeometry = new THREE.PlaneGeometry(200, 200);
            const floorMaterial = new THREE.MeshStandardMaterial({
                color: 0x87CEEB,
                side: THREE.DoubleSide,
                roughness: 0.5,
                metalness: 0.2
            });
            const floor = new THREE.Mesh(floorGeometry, floorMaterial);
            floor.rotation.x = -Math.PI / 2;
            floor.position.set(0, 0, 0);
            floor.name = 'floor'; // Name it so we can identify it
            scene.add(floor);

            // Walls
            const wallMaterialSettings = {
                side: THREE.DoubleSide,
                roughness: 0.7,
                metalness: 0.1
            };
            
            const wall1 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xffffff, ...wallMaterialSettings })
            );
            wall1.position.set(0, 50, -100);
            wall1.name = 'wall';
            scene.add(wall1);

            const wall2 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xffffff, ...wallMaterialSettings })
            );
            wall2.rotation.y = Math.PI / 2;
            wall2.position.set(100, 50, 0);
            wall2.name = 'wall';
            scene.add(wall2);

            const wall3 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xffffff, ...wallMaterialSettings })
            );
            wall3.rotation.y = Math.PI / 2;
            wall3.position.set(-100, 50, 0);
            wall3.name = 'wall';
            scene.add(wall3);

            const wall4 = new THREE.Mesh(
                new THREE.BoxGeometry(200, 100, 4), 
                new THREE.MeshStandardMaterial({ color: 0xffffff, ...wallMaterialSettings })
            );
            wall4.position.set(0, 50, 100);
            wall4.name = 'wall';
            scene.add(wall4);

            const walls = [
                { mesh: wall1, normal: new THREE.Vector3(0, 0, 1) },     // Back wall 
                { mesh: wall2, normal: new THREE.Vector3(-1, 0, 0) },    // Right wall
                { mesh: wall3, normal: new THREE.Vector3(1, 0, 0) },     // Left wall
                { mesh: wall4, normal: new THREE.Vector3(0, 0, -1) }     // Front wall
            ];

            this.walls = walls;

            // Lighting
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
            scene.add(ambientLight);

            const fillLight = new THREE.DirectionalLight(0xffffff, 1);
            fillLight.position.set(20, 100, 20);
            scene.add(fillLight);
            this.fillLight = fillLight;

            // Camera
            camera.position.set(0, 100, 100);
            camera.lookAt(0, 50, 0);

            // Controls
            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI/2;
            controls.minDistance = 260;
            controls.maxDistance = 500;
            this.controls = controls;

            // Model Loading with automatic scaling
            const gltfLoader = new GLTFLoader();
            
            const loadModel = (path, position, targetSize, rotation, name, callback) => {
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
                    
                    // Set name and make draggable
                    model.name = name;
                    model.userData.isDraggable = true;
                    model.userData.originalPosition = model.position.clone();
                    
                    model.traverse((child) => {
                        if (child.isMesh) {
                            child.castShadow = true;
                            child.receiveShadow = true;
                            child.userData.parent = model; // Reference to parent model
                        }
                    });
                    
                    scene.add(model);
                    
                    // Add to draggable objects
                    this.draggableObjects.push(model);
                    
                    if (callback) callback(model);
                    
                    console.log(`Loaded ${path}:`, 
                        `Original size: ${currentSize.x.toFixed(2)} x ${currentSize.y.toFixed(2)} x ${currentSize.z.toFixed(2)}, 
                         Scale factor: ${uniformScale.toFixed(2)}`);
                         
                }, undefined, (error) => {
                    console.error('Error loading model:', error);
                });
            };  

            // Load draggable models
            loadModel('models/bath.glb', 
                { x: 50, y: 25, z: -70 }, 
                { x: 150, y: 50, z: 70 }, 
                { x: 0, y: 0, z: 0 },
                'bath'
            );

            loadModel('models/door.glb', 
                { x: 80, y: 37, z: 98 }, 
                { x: 5, y: 200, z: 100 }, 
                { x: 0, y: Math.PI / 2, z: 0 },
                'door'
            );

            loadModel('models/mirror.glb', 
                { x: -100, y: 60, z: 0 }, 
                { x: -60, y: 80, z: 45 }, 
                { x: 0, y: Math.PI/2, z: 0 },
                'mirror'
            );

            loadModel('models/toilet.glb', 
                { x: -50, y: 10, z: -60 }, 
                { x: 40, y: 70, z: 60 }, 
                { x: 0, y: Math.PI / 70, z: 0 },
                'toilet'
            );

            // Add mouse event listeners for dragging
            this.addDragListeners();

            // Animation
            const animate = () => {
                requestAnimationFrame(animate);
                
                const cameraPosition = new THREE.Vector3();
                camera.getWorldPosition(cameraPosition);  

                const cameraDirection = new THREE.Vector3();
                camera.getWorldDirection(cameraDirection);

                this.walls.forEach(({ mesh, normal }) => {
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
                
                this.fillLight.position.copy(camera.position);
                this.fillLight.position.y += 50; // Keep light above camera
                
                if (!this.isDragging) {
                    this.controls.update();
                }
                
                renderer.render(scene, camera);
            };

            animate();

            // Window resize handler
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });
        },

        addDragListeners() {
            const container = this.renderer.domElement;
            
            // Mouse down event
            container.addEventListener('mousedown', (event) => {
                event.preventDefault();
                this.onMouseDown(event);
            });

            // Mouse move event
            container.addEventListener('mousemove', (event) => {
                event.preventDefault();
                this.onMouseMove(event);
            });

            // Mouse up event
            container.addEventListener('mouseup', (event) => {
                event.preventDefault();
                this.onMouseUp(event);
            });

            // Prevent context menu
            container.addEventListener('contextmenu', (event) => {
                event.preventDefault();
            });
        },

        onMouseDown(event) {
            this.updateMousePosition(event);
            
            // Cast ray from camera through mouse position
            this.raycaster.setFromCamera(this.mouse, this.camera);
            
            // Check for intersections with draggable objects
            const intersects = this.raycaster.intersectObjects(this.draggableObjects, true);
            
            if (intersects.length > 0) {
                // Find the parent draggable object
                let targetObject = intersects[0].object;
                while (targetObject.parent && !targetObject.userData.isDraggable) {
                    targetObject = targetObject.parent;
                }
                
                if (targetObject.userData.isDraggable) {
                    this.isDragging = true;
                    this.dragObject = targetObject;
                    this.controls.enabled = false; // Disable orbit controls while dragging
                    
                    // Calculate drag plane (horizontal plane at object's Y position)
                    const objectY = this.dragObject.position.y;
                    this.dragPlane.setFromNormalAndCoplanarPoint(
                        new THREE.Vector3(0, 1, 0),
                        new THREE.Vector3(0, objectY, 0)
                    );
                    
                    // Calculate offset from intersection point to object center
                    const intersection = new THREE.Vector3();
                    this.raycaster.ray.intersectPlane(this.dragPlane, intersection);
                    this.dragOffset.subVectors(this.dragObject.position, intersection);
                    
                    // Visual feedback - slightly lift the object
                    this.dragObject.position.y += 2;
                    
                    this.renderer.domElement.style.cursor = 'grabbing';
                }
            }
        },

        onMouseMove(event) {
            this.updateMousePosition(event);
            
            if (this.isDragging && this.dragObject) {
                // Cast ray and find intersection with drag plane
                this.raycaster.setFromCamera(this.mouse, this.camera);
                const intersection = new THREE.Vector3();
                
                if (this.raycaster.ray.intersectPlane(this.dragPlane, intersection)) {
                    // Add offset to get the desired position
                    const newPosition = intersection.add(this.dragOffset);
                    
                    // Constrain to room boundaries
                    newPosition.x = Math.max(this.roomBounds.minX, Math.min(this.roomBounds.maxX, newPosition.x));
                    newPosition.z = Math.max(this.roomBounds.minZ, Math.min(this.roomBounds.maxZ, newPosition.z));
                    
                    // Update object position
                    this.dragObject.position.x = newPosition.x;
                    this.dragObject.position.z = newPosition.z;
                }
            } else {
                // Check if hovering over draggable object for cursor feedback
                this.raycaster.setFromCamera(this.mouse, this.camera);
                const intersects = this.raycaster.intersectObjects(this.draggableObjects, true);
                
                if (intersects.length > 0) {
                    let targetObject = intersects[0].object;
                    while (targetObject.parent && !targetObject.userData.isDraggable) {
                        targetObject = targetObject.parent;
                    }
                    
                    if (targetObject.userData.isDraggable) {
                        this.renderer.domElement.style.cursor = 'grab';
                    }
                } else {
                    this.renderer.domElement.style.cursor = 'default';
                }
            }
        },

        onMouseUp() {
            if (this.isDragging && this.dragObject) {
                // Lower the object back to its original Y position
                this.dragObject.position.y -= 2;
                
                this.isDragging = false;
                this.dragObject = null;
                this.controls.enabled = true; // Re-enable orbit controls
                
                this.renderer.domElement.style.cursor = 'default';
            }
        },

        updateMousePosition(event) {
            const rect = this.renderer.domElement.getBoundingClientRect();
            this.mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
            this.mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
        }
    }
}
</script>

<style scoped>
.three-container {
    padding: 0;
    margin: 0;
    cursor: default;
}
</style>