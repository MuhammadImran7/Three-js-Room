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
      draggableObjects: [],   // top-level draggable models
      pickTargets: [],        // simple invisible boxes for raycast picking
      isDragging: false,
      dragObject: null,

      mouse: new THREE.Vector2(),
      raycaster: new THREE.Raycaster(),
      dragPlane: new THREE.Plane(),
      dragOffset: new THREE.Vector3(),

      roomBounds: { minX: -95, maxX: 95, minZ: -95, maxZ: 95, minY: 50, maxY: 100 },
      walls: [],
      wallBoxes: [],

      // temps / caches
      tmpVec3: new THREE.Vector3(),
      cameraPos: new THREE.Vector3(),
      cameraDir: new THREE.Vector3(),
      wallPos:   new THREE.Vector3(),
      camToWall: new THREE.Vector3(),
      lastCamMatrix: new THREE.Matrix4(),
    };
  },
  mounted() {
    this.initThreeJS();
  },
  methods: {
    initThreeJS() {
      // Scene / camera / renderer
      const scene = new THREE.Scene();
      const camera = new THREE.PerspectiveCamera(80, window.innerWidth / window.innerHeight, 0.1, 1000);

      const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
      // No shadow map (faster). If you need shadows later, enable and tune.
    //    renderer.shadowMap.enabled = true;
      this.$refs.threeContainer.appendChild(renderer.domElement);
      scene.background = new THREE.Color(0xeeeeee);

      this.scene = scene;
      this.camera = camera;
      this.renderer = renderer;

      // Floor (cheaper material)
      const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(200, 200),
        new THREE.MeshLambertMaterial({ color: 0x000000, side: THREE.DoubleSide })
      );
      floor.rotation.x = -Math.PI / 2;
      floor.position.set(0, 0, 0);
      floor.name = 'floor';
      scene.add(floor);

      // Walls (cheaper material)
      const wallMat = new THREE.MeshLambertMaterial({ color: 0xffffff });
      const wall1 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 1), wallMat.clone());
      wall1.position.set(0, 50, -100); wall1.name = 'wall'; scene.add(wall1);

      const wall2 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 1), wallMat.clone());
      wall2.rotation.y = Math.PI / 2; wall2.position.set(100, 50, 0); wall2.name = 'wall'; scene.add(wall2);

      const wall3 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 1), wallMat.clone());
      wall3.rotation.y = Math.PI / 2; wall3.position.set(-100, 50, 0); wall3.name = 'wall'; scene.add(wall3);

      const wall4 = new THREE.Mesh(new THREE.BoxGeometry(200, 100, 1), wallMat.clone());
      wall4.position.set(0, 50, 100); wall4.name = 'wall'; scene.add(wall4);

      this.walls = [
        { mesh: wall1, normal: new THREE.Vector3(0, 0, 1) },
        { mesh: wall2, normal: new THREE.Vector3(-1, 0, 0) },
        { mesh: wall3, normal: new THREE.Vector3(1, 0, 0) },
        { mesh: wall4, normal: new THREE.Vector3(0, 0, -1) }
      ];
      // Precompute static wall boxes once
      this.wallBoxes = this.walls.map(({ mesh }) => new THREE.Box3().setFromObject(mesh));

      // Lights
      scene.add(new THREE.AmbientLight(0xffffff, 1));
      const fillLight = new THREE.DirectionalLight(0xffffff, 1);
      fillLight.position.set(20, 100, 20);
      scene.add(fillLight);
      this.fillLight = fillLight;

      // Camera & controls
      camera.position.set(0, 120, 180);
      camera.lookAt(0, 50, 0);

      const controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;          // still smooth, but we render on demand
      controls.dampingFactor = 0.15;
      controls.screenSpacePanning = false;
      controls.maxPolarAngle = Math.PI / 2;
      controls.minDistance = 200;
      controls.maxDistance = 480;
      this.controls = controls;

      // Render-on-demand: render when controls change
      controls.addEventListener('change', this.render);

      // Loader
      const gltfLoader = new GLTFLoader();

      const loadModel = (path, position, targetSize, rotation, name) => {
        gltfLoader.load(path, (gltf) => {
          const model = gltf.scene;

          // size & scale
          const box = new THREE.Box3().setFromObject(model);
          const size = box.getSize(new THREE.Vector3());
          const s = Math.min(targetSize.x / size.x, targetSize.y / size.y, targetSize.z / size.z);

          model.scale.set(s, s, s);
          model.position.set(position.x, position.y, position.z);
          model.rotation.set(rotation.x, rotation.y, rotation.z);

          // center to given position
          const scaledBox = new THREE.Box3().setFromObject(model);
          const center = scaledBox.getCenter(new THREE.Vector3());
          model.position.sub(center).add(new THREE.Vector3(position.x, position.y, position.z));

          model.name = name;
          model.userData.isDraggable = true;
          model.userData.originalPosition = model.position.clone();

          // Optional: don’t set cast/receiveShadow unless you enable renderer.shadowMap
          model.traverse((child) => {
            if (child.isMesh) {
              child.userData.parent = model;
              child.frustumCulled = true;
            }
          });

          // Cache a local-space bounding box and a reusable worldBox
          model.updateWorldMatrix(true, true);
          const worldBox = new THREE.Box3().setFromObject(model);
          const inv = new THREE.Matrix4().copy(model.matrixWorld).invert();
          model.userData.localBox = worldBox.clone().applyMatrix4(inv); // local-space
          model.userData.worldBox = new THREE.Box3();                   // holder

          // Build a simple invisible pick proxy (single box) for raycasting
          const lb = model.userData.localBox; // min/max in local space
          const sizeL = new THREE.Vector3().subVectors(lb.max, lb.min);
          const centerL = new THREE.Vector3().addVectors(lb.min, lb.max).multiplyScalar(0.5);

          const pickGeo = new THREE.BoxGeometry(sizeL.x, sizeL.y, sizeL.z);
          const pickMat = new THREE.MeshBasicMaterial({ visible: false });
          const pickProxy = new THREE.Mesh(pickGeo, pickMat);
          pickProxy.position.copy(centerL);
          pickProxy.name = `${name}_pickProxy`;

          // Parent the pick proxy to the model so it follows transforms
          model.add(pickProxy);

          this.scene.add(model);
          this.draggableObjects.push(model);
          this.pickTargets.push(pickProxy);

          this.render();
        }, undefined, (e) => console.error('Error loading model:', e));
      };

      // Load your models
      loadModel('models/bath.glb',   { x:  50, y: 25, z: -70 }, { x: 150, y:  50, z:  70 }, { x: 0, y: 0,             z: 0 }, 'bath');
      loadModel('models/door.glb',   { x:  80, y: 37, z:  98 }, { x:   5, y: 200, z: 100 }, { x: 0, y: Math.PI / 2,  z: 0 }, 'door');
      loadModel('models/mirror.glb', { x: 100, y: 60, z:   0 }, { x:  60, y:  80, z:  45 }, { x: 0, y: Math.PI / 2,  z: 0 }, 'mirror');
      loadModel('models/toilet.glb', { x:  -50, y: 10, z: -60 }, { x:  40, y:  70, z:  60 }, { x: 0, y: Math.PI / 70, z: 0 }, 'toilet');

      // Events
      this.addDragListeners();

      // First render
      this.render();

      // Resize
      window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
        this.render();
      });
    },

    // ---------- Render-on-demand ----------
    render() {
      // update wall visibility if camera changed
      const cam = this.camera;
      if (!this.lastCamMatrix.equals(cam.matrixWorld)) {
        this.lastCamMatrix.copy(cam.matrixWorld);

        cam.getWorldPosition(this.cameraPos);
        cam.getWorldDirection(this.cameraDir);

        for (const { mesh, normal } of this.walls) {
          mesh.getWorldPosition(this.wallPos);
          this.camToWall.subVectors(this.wallPos, this.cameraPos).normalize();
          mesh.visible = this.camToWall.dot(normal) <= 0.1;
        }
      }

      // keep light following camera (cheap)
      this.fillLight.position.copy(this.camera.position).add(new THREE.Vector3(0, 50, 0));

      this.renderer.render(this.scene, this.camera);
    },

    // ---------- Events (pointer) ----------
    addDragListeners() {
      const el = this.renderer.domElement;
      el.addEventListener('pointerdown', this.onPointerDown);
      el.addEventListener('pointermove', this.onPointerMove);
      el.addEventListener('pointerup',   this.onPointerUp);
      el.addEventListener('contextmenu', (e) => e.preventDefault());
    },

    onPointerDown(event) {
      this.updateMousePosition(event);
      this.raycaster.setFromCamera(this.mouse, this.camera);

      // Raycast only against simple pick proxies (fast)
      const hits = this.raycaster.intersectObjects(this.pickTargets, false);
      if (!hits.length) return;

      // Draggable root is the proxy's parent (the model)
      let target = hits[0].object.parent;
      if (!target || !target.userData.isDraggable) return;

      this.isDragging = true;
      this.dragObject = target;
      this.controls.enabled = false;

      // Drag plane @ object's current Y
      const y = this.dragObject.position.y;
      this.dragPlane.setFromNormalAndCoplanarPoint(
        new THREE.Vector3(0, 1, 0),
        new THREE.Vector3(0, y, 0)
      );

      // Offset (object center - hit point)
      const intersection = new THREE.Vector3();
      this.raycaster.ray.intersectPlane(this.dragPlane, intersection);
      this.dragOffset.subVectors(this.dragObject.position, intersection);

      this.renderer.domElement.style.cursor = 'grabbing';
      this.render();
    },

    onPointerMove(event) {
      this.updateMousePosition(event);

      if (this.isDragging && this.dragObject) {
        this.raycaster.setFromCamera(this.mouse, this.camera);

        const intersection = this.tmpVec3;
        if (this.raycaster.ray.intersectPlane(this.dragPlane, intersection)) {
          const desiredX = intersection.x + this.dragOffset.x;
          const desiredZ = intersection.z + this.dragOffset.z;

          const clampedX = Math.max(this.roomBounds.minX, Math.min(this.roomBounds.maxX, desiredX));
          const clampedZ = Math.max(this.roomBounds.minZ, Math.min(this.roomBounds.maxZ, desiredZ));

          const obj = this.dragObject;
          const oldX = obj.position.x, oldZ = obj.position.z;

          obj.position.x = clampedX;
          obj.position.z = clampedZ;

          if (this.intersectsAny(obj)) {
            // revert and nudge
            obj.position.x = oldX;
            obj.position.z = oldZ;

            const dx = clampedX - oldX, dz = clampedZ - oldZ;
            const len = Math.hypot(dx, dz);
            if (len > 1e-6) {
              const eps = 0.5;
              const nx = oldX + (dx / len) * Math.max(0, len - eps);
              const nz = oldZ + (dz / len) * Math.max(0, len - eps);
              obj.position.x = nx;
              obj.position.z = nz;
              if (this.intersectsAny(obj)) {
                obj.position.x = oldX;
                obj.position.z = oldZ;
              }
            }
          }
          this.render(); // draw only when something actually moved
        }
      } else {
        // Hover feedback (cheap because we raycast against pick proxies)
        this.raycaster.setFromCamera(this.mouse, this.camera);
        const hits = this.raycaster.intersectObjects(this.pickTargets, false);
        this.renderer.domElement.style.cursor = hits.length ? 'grab' : 'default';
        // No need to render on hover unless you change visuals
      }
    },

    onPointerUp() {
      if (!this.isDragging) return;
      this.isDragging = false;
      this.dragObject = null;
      this.controls.enabled = true;
      this.renderer.domElement.style.cursor = 'default';
      this.render();
    },

    // ---------- Helpers ----------
    updateMousePosition(event) {
      const rect = this.renderer.domElement.getBoundingClientRect();
      this.mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      this.mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    },

    // Fast world box using cached local box (no traversal)
    getWorldBoxFast(obj) {
      obj.updateWorldMatrix(true, false);
      return obj.userData.worldBox.copy(obj.userData.localBox).applyMatrix4(obj.matrixWorld);
    },

    intersectsAny(targetObj) {
      const targetBox = this.getWorldBoxFast(targetObj);

      // Other draggables (if they don't move, you could precompute their world boxes once)
      for (const other of this.draggableObjects) {
        if (other === targetObj) continue;
        const otherBox = this.getWorldBoxFast(other);
        if (targetBox.intersectsBox(otherBox)) return true;
      }

      // Static walls
      for (const wallBox of this.wallBoxes) {
        if (targetBox.intersectsBox(wallBox)) return true;
      }
      return false;
    },
  }
};
</script>

<style scoped>
.three-container {
  padding: 0;
  margin: 0;
  cursor: default;
}
</style>
