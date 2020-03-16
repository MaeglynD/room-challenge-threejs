<template>
  <v-app>
	<div ref="container"></div>
	<v-btn @click="update_cube()">text</v-btn>
  </v-app>
</template>

<script>
// import HelloWorld from './components/HelloWorld';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

let plane;
let mouse;
let raycaster;
let isShiftDown = false;
let rollOverMesh;
let rollOverGeo;
let rollOverMaterial;
let cubeMaterial;

const objects = [];

export default {
	name: 'App',
	data: () => ({
		camera: null,
		scene: null,
		renderer: null,
		controls: null,
		mesh: null,
		multiplier: 1,
	}),
	mounted() {
		// INIT
		// const geometry = new THREE.BoxGeometry(5, 5, 5);
		// const material = new THREE.MeshLambertMaterial({
		// 	color: 0xffffff,
		// 	opacity: 0.5,
		// 	transparent: true,
		// });

		// const group = new THREE.Group();
		// this.scene.add(group);

		// this.mesh = new THREE.Mesh(geometry, material);
		// this.mesh.material.side = THREE.BackSide; // back faces
		// this.mesh.renderOrder = 0;
		// group.add(this.mesh);

		// this.mesh2 = new THREE.Mesh(geometry, material.clone());
		// this.mesh2.material.side = THREE.FrontSide; // front faces
		// this.mesh2.renderOrder = 1;
		// group.add(this.mesh2);

		// this.camera.position.set(15, 20, 30);

		// this.scene.add(new THREE.AxesHelper(20));

		// this.scene.add(new THREE.AmbientLight(0x222222));
		// const light = new THREE.PointLight(0xffffff, 1);
		// this.camera.add(light);

		// this.renderer = new THREE.WebGLRenderer({ antialias: true });
		// this.renderer.setSize(window.innerWidth, window.innerHeight);
		// this.controls = new OrbitControls(this.camera, this.renderer.domElement);
		// this.$refs.container.appendChild(this.renderer.domElement);

		// // // START
		// this.animate();

		this.init();
		this.render();
	},
	methods: {
		init() {
			this.camera = new THREE.PerspectiveCamera(
				45, window.innerWidth / window.innerHeight, 1, 10000,
			);
			this.camera.position.set(500, 800, 1300);
			this.camera.lookAt(0, 0, 0);

			this.scene = new THREE.Scene();
			this.scene.background = new THREE.Color(0x202224);

			// roll-over helpers
			rollOverGeo = new THREE.BoxBufferGeometry(25, 25, 25);
			rollOverMaterial = new THREE.MeshBasicMaterial({
				color: 0xff0000, opacity: 0.5, transparent: true,
			});
			rollOverMesh = new THREE.Mesh(rollOverGeo, rollOverMaterial);
			this.scene.add(rollOverMesh);

			// cubes
			this.cubeGeo = new THREE.BoxBufferGeometry(25, 25, 25);
			cubeMaterial = new THREE.MeshLambertMaterial({
				color: 0x2e79ce,
			});

			// grid
			const gridHelper = new THREE.GridHelper(1000, 40);
			this.scene.add(gridHelper);

			raycaster = new THREE.Raycaster();
			mouse = new THREE.Vector2();

			const geometry = new THREE.PlaneBufferGeometry(1000, 1000);
			geometry.rotateX(-Math.PI / 2);

			plane = new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ visible: false }));
			this.scene.add(plane);

			objects.push(plane);

			// lights
			const ambientLight = new THREE.AmbientLight(0x606060);
			this.scene.add(ambientLight);

			const directionalLight = new THREE.DirectionalLight(0xffffff);
			directionalLight.position.set(1, 0.75, 0.5).normalize();
			this.scene.add(directionalLight);

			this.renderer = new THREE.WebGLRenderer({ antialias: true });
			this.renderer.setPixelRatio(window.devicePixelRatio);
			this.renderer.setSize(window.innerWidth, window.innerHeight);
			this.$refs.container.appendChild(this.renderer.domElement);

			this.controls = new OrbitControls(this.camera, this.renderer.domElement);

			document.addEventListener('mousemove', this.onDocumentMouseMove, false);
			document.addEventListener('mousedown', this.onDocumentMouseDown, false);
			document.addEventListener('keydown', this.onDocumentKeyDown, false);
			document.addEventListener('keyup', this.onDocumentKeyUp, false);

			window.addEventListener('resize', this.onWindowResize, false);
		},
		// animate() {
		// 	requestAnimationFrame(this.animate);
		// 	this.mesh.rotation.x += 0.001;
		// 	this.mesh.rotation.y += 0.001;
		// 	this.renderer.render(this.scene, this.camera);
		// 	this.controls.update();
		// 	// debugger;
		// },
		onWindowResize() {
			this.camera.aspect = window.innerWidth / window.innerHeight;
			this.camera.updateProjectionMatrix();

			this.renderer.setSize(window.innerWidth, window.innerHeight);
		},
		onDocumentMouseMove(event) {
			event.preventDefault();

			mouse.set(
				(event.clientX / window.innerWidth) * 2 - 1,
				-(event.clientY / window.innerHeight) * 2 + 1,
			);

			raycaster.setFromCamera(mouse, this.camera);

			const intersects = raycaster.intersectObjects(objects);

			if (intersects.length > 0) {
				const intersect = intersects[0];

				rollOverMesh.position.copy(intersect.point).add(intersect.face.normal);
				rollOverMesh.position.divideScalar(25 * this.multiplier).floor()
					.multiplyScalar(25 * this.multiplier).addScalar((25 * this.multiplier) / 2);
			}

			this.render();
		},
		onDocumentMouseDown(event) {
			event.preventDefault();

			if (event.which === 3) {
				mouse.set(
					(event.clientX / window.innerWidth) * 2 - 1,
					-(event.clientY / window.innerHeight) * 2 + 1,
				);

				raycaster.setFromCamera(mouse, this.camera);

				const intersects = raycaster.intersectObjects(objects);

				if (intersects.length > 0) {
					const intersect = intersects[0];

					// delete cube

					if (isShiftDown) {
						if (intersect.object !== plane) {
							this.scene.remove(intersect.object);

							objects.splice(objects.indexOf(intersect.object), 1);
						}

						// create cube
					} else {
						const v = this.multiplier * 25;
						const voxel = new THREE.Mesh(new THREE.BoxBufferGeometry(v, v, v), cubeMaterial);
						voxel.position.copy(intersect.point).add(intersect.face.normal);
						voxel.position.divideScalar(25 * this.multiplier)
							.floor().multiplyScalar(25 * this.multiplier).addScalar((25 * this.multiplier) / 2);
						this.scene.add(voxel);
						objects.push(voxel);
					}

					this.render();
				}
			}
		},
		onDocumentKeyDown(event) {
			switch (event.keyCode) {
			case 16: isShiftDown = true; break;
			default: break;
			}
		},
		onDocumentKeyUp(event) {
			switch (event.keyCode) {
			case 16: isShiftDown = false; break;
			default: break;
			}
		},
		render() {
			this.renderer.render(this.scene, this.camera);
		},
		update_cube() {
			this.multiplier += 1;
			// const v = this.multiplier * 25;

			rollOverGeo.scale(2, 2, 2);
			// rollOverGeo.parameters.width = v;
			// rollOverGeo.parameters.height = v;
			// rollOverGeo.parameters.depth = v;
			// rollOverGeo.verticesNeedUpdate = true;
		},
	},
};
</script>

<style lang="scss">
@import url('./assets/global.scss');

canvas{
	height: 95%;
	width: 100%;
}
</style>
