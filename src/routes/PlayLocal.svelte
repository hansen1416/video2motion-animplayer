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

	function animate() {
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
		// threeScene.scene.position.set(0, -1, 0);
		threeScene.scene.position.set(0, -100, 0);

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

			console.log(chunks);
		};

		reader.readAsArrayBuffer(file);
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
