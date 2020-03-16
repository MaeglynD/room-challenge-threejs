<template>
	<v-app>
		<v-navigation-drawer
			:mini-variant=true
			:clipped=true
			v-model="drawer"
			right
			floating
			app
			mobile-break-point="0"
			mini-variant-width="80"
			color="#3b322c"
		>
        <v-list three-line>

			<v-list-item class="d-item">
				<v-list-item-avatar size="35">
					<v-img :src="require('@/assets/logo.png')" ></v-img>
				</v-list-item-avatar>
			</v-list-item>

            <v-divider class="mx-3"></v-divider>

            <v-tooltip left transition="slide-x-reverse-transition" color="#3b322c">
				<template v-slot:activator="{ on }">
					<v-list-item slot="activator" v-on="on" class="d-item" height="88" @click="nothing()">
						<v-icon>mdi-cog</v-icon>
					</v-list-item>
				</template>
                <span>settings</span>
            </v-tooltip>

            <v-tooltip left color="#3b322c" transition="slide-x-reverse-transition">
				<template v-slot:activator="{ on }">
					<v-list-item slot="activator" v-on="on" class="d-item" height="88"
						@click="confirmDialog = true">
						<v-icon>mdi-refresh</v-icon>
					</v-list-item>
				</template>
                <span>refresh</span>
            </v-tooltip>

            <v-tooltip left color="#3b322c" transition="slide-x-reverse-transition">
				<template v-slot:activator="{ on }">
					<v-list-item slot="activator" v-on="on"  class="d-item" height="88"
					@click="isFullscreen ? closeFullscreen() : openFullScreen()">
						<v-icon>mdi-fullscreen</v-icon>
					</v-list-item>
				</template>
                <span>fullscreen</span>
            </v-tooltip>

            <v-divider class="mx-3"></v-divider>

            <v-tooltip v-if="$vuetify.breakpoint.smAndUp" left color="#3b322c"
				transition="slide-x-reverse-transition">
				<template v-slot:activator="{ on }" v-on="on">
					<v-list-item slot="activator" class="d-item" height="88"
					@click="infoDialog = true">
						<v-icon>mdi-information</v-icon>
					</v-list-item>
				</template>
                <span>toggle fullscreen</span>
            </v-tooltip>
        </v-list>
    </v-navigation-drawer>

	<div ref="container"></div>

	<v-card class="d-card d-results" min-width="330" color="#3d322d" v-draggable="draggableValue">
		<v-card-title>
			results
		</v-card-title>
		<v-card-text>
			<div class="d-results-a">{{ paint }} <span>litres of paint</span></div>
			<div class="d-results-a">{{ volume }} <span>m² volume</span></div>
			<div class="d-results-a">{{ floor }} <span>m² floor area</span></div>
		</v-card-text>
	</v-card>

	<v-dialog
		v-model="confirmDialog" :overlay="false"
		max-width="500px"
		transition="dialog-transition"
	>
		<v-card class="d-card" color="#3d322d">
			<v-card-title>
				Are you sure you want to reload the scene?
			</v-card-title>
			<v-card-text>
				All objects will be removed from the scene... forever.
			</v-card-text>
			<v-divider></v-divider>
			<v-card-actions>
				<v-spacer></v-spacer>
				<v-btn text @click="confirmDialog = false">cancel</v-btn>
				<v-btn text @click="removeGeometry()">confirm</v-btn>
			</v-card-actions>
		</v-card>
	</v-dialog>

	<v-dialog
		v-model="infoDialog" :overlay="false"
		max-width="500px"
		transition="dialog-transition"
	>
		<v-card class="d-card" color="#3d322d">
			<v-card-title>
				Information
			</v-card-title>
			<v-card-text>
				Left click to place a block <br>
				Shift + left click to remove a block
			</v-card-text>
			<v-divider></v-divider>
			<v-card-actions>
				<v-spacer></v-spacer>
				<v-btn text @click="infoDialog = false">close</v-btn>
			</v-card-actions>
		</v-card>
	</v-dialog>
</v-app>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { Draggable } from 'draggable-vue-directive';

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
	directives: {
		Draggable,
	},
	data: () => ({
		camera: null,
		scene: null,
		renderer: null,
		controls: null,
		mesh: null,
		multiplier: 1,
		drawer: true,
		draggableValue: {},
		confirmDialog: false,
		paint: 0,
		volume: 0,
		floor: 0,
		isFullscreen: false,
		infoDialog: false,
	}),
	created() {
		this.$vuetify.theme.dark = true;
	},
	mounted() {
		this.init();
		this.render();
		this.draggableValue.boundingElement = this.renderer.domElement;
	},
	methods: {
		init() {
			this.camera = new THREE.PerspectiveCamera(
				45, window.innerWidth / window.innerHeight, 1, 10000,
			);
			this.camera.position.set(500, 800, 1300);
			this.camera.lookAt(0, 0, 0);

			this.scene = new THREE.Scene();
			this.scene.background = new THREE.Color(0x251d17);

			// roll-over helpers
			rollOverGeo = new THREE.BoxGeometry(25, 25, 25);
			rollOverMaterial = new THREE.MeshBasicMaterial({
				color: 0xff0000, opacity: 0.5, transparent: true,
			});
			rollOverMesh = new THREE.Mesh(rollOverGeo, rollOverMaterial);
			this.scene.add(rollOverMesh);

			// cubes
			this.cubeGeo = new THREE.BoxGeometry(25, 25, 25);
			cubeMaterial = new THREE.MeshLambertMaterial({
				color: 0x48E5C2,
			});

			// grid
			const gridHelper = new THREE.GridHelper(1000, 40);
			this.scene.add(gridHelper);

			raycaster = new THREE.Raycaster();
			mouse = new THREE.Vector2();

			const geometry = new THREE.PlaneGeometry(1000, 1000);
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
						const voxel = new THREE.Mesh(new THREE.BoxGeometry(v, v, v), cubeMaterial);
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
			rollOverGeo.scale(2, 2, 2);
		},
		nothing() {},
		removeGeometry() {
			objects.slice(1).forEach((x) => {
				this.scene.remove(x);
			});
			this.confirmDialog = false;
		},
		updateValues() {
			// const geoObj = new THREE.Geometry();
			// objects.slice(1).forEach((x) => {
			// 	geoObj.merge(x.geometry, x.matrix);
			// });
			// geoObj.verticesNeedUpdate = true;
			// geoObj.mergeVertices();
			// geoObj.computeVertexNormals();
			// geoObj.computeFaceNormals();
			// // geoObj.faces.map((x) => {

			// // })
			// const testmesh = new THREE.Mesh(geoObj, new THREE.MeshBasicMaterial({
			// 	color: 0x0000ff, opacity: 0.3, transparent: true,
			// }));
			// this.scene.add(testmesh);
			// console.log(testmesh);
		},
		openFullScreen() {
			const elem = document.documentElement;

			if (elem.requestFullscreen) {
				elem.requestFullscreen();
			} else if (elem.mozRequestFullScreen) {
				elem.mozRequestFullScreen();
			} else if (elem.webkitRequestFullscreen) {
				elem.webkitRequestFullscreen();
			} else if (elem.msRequestFullscreen) {
				elem.msRequestFullscreen();
			}
			this.isFullscreen = !this.isFullscreen;
		},
		closeFullscreen() {
			if (document.exitFullscreen) {
				document.exitFullscreen();
			} else if (document.mozCancelFullScreen) {
				document.mozCancelFullScreen();
			} else if (document.webkitExitFullscreen) {
				document.webkitExitFullscreen();
			} else if (document.msExitFullscreen) {
				document.msExitFullscreen();
			}
			this.isFullscreen = !this.isFullscreen;
		},
	},
};
</script>

<style lang="scss">
@import './assets/global.scss';
</style>
