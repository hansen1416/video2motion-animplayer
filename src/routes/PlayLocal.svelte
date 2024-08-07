<script>
	/**
	 * load all animation fbx files, and send animation clip json to "http://localhost:2020"
	 * which is a nodejs server, where the json files will be saved
	 */
	import _ from "lodash";
	import { onDestroy, onMount } from "svelte";
	import * as THREE from "three";
	import ThreeScene from "../lib/ThreeScene";
	import { loadGLTF } from "../utils/ropes";

	/** @type {HTMLCanvasElement} */
	let canvas;

	/** @type {ThreeScene} */
	let threeScene;

	let animation_pointer = 0;

	let bones = {};

	/** @type {HTMLInputElement} */
	let fileInput;

	let animationData;

	let animationFrame = 0;

	const joints_mapping = {
		0: "Hips",
		1: "LeftUpLeg",
		2: "RightUpLeg",
		3: "Spine",
		4: "LeftLeg",
		5: "RightLeg",
		6: "Spine1",
		7: "LeftFoot",
		8: "RightFoot",
		9: "Spine2",
		10: "LeftToeBase",
		11: "RightToeBase",
		12: "Neck",
		13: "LeftShoulder",
		14: "RightShoulder",
		15: "Head",
		16: "LeftArm",
		17: "RightArm",
		18: "LeftForeArm",
		19: "RightForeArm",
		20: "LeftHand",
		21: "RightHand",
		22: "L_Hand",
		23: "R_Hand",
	};

	// [
	// 	"Hips",
	// 	"Spine",
	// 	"Spine1",
	// 	"Spine2",
	// 	"Neck",
	// 	"Head",

	// 	"LeftShoulder",
	// 	"LeftArm",
	// 	"LeftForeArm",
	// 	"LeftHand",
	// 	"RightShoulder",
	// 	"RightArm",
	// 	"RightForeArm",
	// 	"RightHand",

	// 	"LeftUpLeg",
	// 	"LeftLeg",
	// 	"LeftFoot",
	// 	"LeftToeBase",
	// 	"LeftToe_End",
	// 	"RightUpLeg",
	// 	"RightLeg",
	// 	"RightFoot",
	// 	"RightToeBase",
	// 	"RightToe_End",
	// ];

	function animate() {
		if (animationData) {
			// todo apply animation data to bones
			applyAnimationData();

			animationFrame += 1;

			if (animationFrame >= animationData.length - 1) {
				animationFrame = 0;
			}
		}

		// update physics world and threejs renderer
		threeScene.onFrameUpdate();

		animation_pointer = requestAnimationFrame(animate);
	}

	onMount(() => {
		threeScene = new ThreeScene(
			canvas,
			document.documentElement.clientWidth,
			document.documentElement.clientHeight,
		);

		// -100 is ground level
		threeScene.scene.position.set(0, -1, 0);
		// threeScene.scene.position.set(0, -100, 0);

		Promise.all([loadGLTF(`/glb/dors.glb`)]).then(([glb_model]) => {
			glb_model = glb_model.scene.children[0];

			glb_model.name = "diva";

			// console.log(fbx_unity_anim);
			threeScene.scene.add(glb_model);

			glb_model.traverse((node) => {
				// @ts-ignore
				if (node.isMesh) {
					node.castShadow = true;
				}
				// @ts-ignore
				if (node.isBone) {
					// @ts-ignore
					if (bones[node.name] === undefined) {
						// somehow maximo has double bones, so only use the first one
						bones[node.name] = node;
					}
				}
			});

			console.log(Object.keys(bones));
		});

		animate();
	});

	onDestroy(() => {
		// unsubscribe all stores
		cancelAnimationFrame(animation_pointer);

		threeScene.dispose();
	});

	function onFileChange(e) {
		const file = e.target.files[0];

		const reader = new FileReader();

		reader.onload = (e) => {
			const arrayBuffer = e.target.result;
			// Use the array buffer to process the binary data
			// console.log(arrayBuffer);

			const data = new Float32Array(arrayBuffer);

			// split the data to chunks, each of length 72
			const chunks = _.chunk(data, 72);

			// console.log(chunks);
			animationData = chunks;
		};

		reader.readAsArrayBuffer(file);
	}

	function applyAnimationData() {
		if (animationData) {
			const data = animationData[animationFrame];

			const data_chunks = _.chunk(data, 3);

			// console.log(data_chunks);

			for (let i = 0; i < data_chunks.length; i++) {
				const bone = bones[joints_mapping[i]];

				if (bone) {
					console.log(data_chunks[i]);

					const [x, y, z] = data_chunks[i];

					bone.rotation.set(x, y, z);
					//  = new THREE.Euler(x, y, z);
				}
			}
		}
	}
</script>

<section>
	<canvas bind:this={canvas} />
</section>

<input type="file" bind:this={fileInput} on:change={onFileChange} />

<style>
	canvas {
		/* this will only affect <canvas> elements in this component */
		z-index: -1;
		position: absolute;
		top: 0;
		left: 0;
		bottom: 0;
		right: 0;
	}
</style>
