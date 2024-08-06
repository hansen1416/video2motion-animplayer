<script>
	/**
	 * load all animation fbx files, and send animation clip json to "http://localhost:2020"
	 * which is a nodejs server, where the json files will be saved
	 */
	import _ from "lodash";
	import { onDestroy, onMount } from "svelte";
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
		0: "Pelvis",
		1: "L_Hip",
		2: "R_Hip",
		3: "Spine1",
		4: "L_Knee",
		5: "R_Knee",
		6: "Spine2",
		7: "L_Ankle",
		8: "R_Ankle",
		9: "Spine3",
		10: "L_Foot",
		11: "R_Foot",
		12: "Neck",
		13: "L_Collar",
		14: "R_Collar",
		15: "Head",
		16: "L_Shoulder",
		17: "R_Shoulder",
		18: "L_Elbow",
		19: "R_Elbow",
		20: "L_Wrist",
		21: "R_Wrist",
		22: "L_Hand",
		23: "R_Hand",
	};

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

			// console.log(data);

			// apply data to bones
			Object.keys(bones).forEach((bone_name, i) => {
				const bone = bones[bone_name];

				const x = data[i * 3];
				const y = data[i * 3 + 1];
				const z = data[i * 3 + 2];

				bone.position.set(x, y, z);
			});
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
