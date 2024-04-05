<script>
	/**
	 *
	 */
	import _ from "lodash";
	import { onDestroy, onMount } from "svelte";
	import * as THREE from "three";
	import ThreeScene from "../lib/ThreeScene";
	import { loadGLTF, loadJSON } from "../utils/ropes";

	/** @type {HTMLCanvasElement} */
	let canvas;

	/** @type {ThreeScene} */
	let threeScene;

	let animation_pointer = 0;

	/** @type {{[key: string]: }} */
	let bones = {};
	let animation_data = null;

	let anim_step = 0;
	let max_anim_step = 0;

	// let animation_path = `/anim-calculated-quaternion/Arms Hip Hop Dance-30-0.json`;
	// let animation_path = `/anim-calculated-quaternion/Walking (9)-30-0.json`;
	let animation_path = `/anim-calculated-quaternion/180 Turn W_ Briefcase (1)-30-0.json`;
	// let animation_path = `/anim-calculated-quaternion/Receiver Catch-30-0.json`;
	// let animation_path = `/anim-calculated-quaternion/Pull Plant-30-0.json`;
	// let animation_path = `/anim-calculated-quaternion/Sitting Clap (4)-30-0.json`;

	let frame_skip = 1;
	let counter = 0;

	function animate() {
		if (counter % frame_skip === 0 && animation_data && bones) {
			for (let bone_name in bones) {
				if (animation_data[bone_name] !== undefined) {
					if (animation_data[bone_name][anim_step] === undefined) {
						continue;
					}

					const x = animation_data[bone_name][anim_step][0];
					const y = animation_data[bone_name][anim_step][1];
					const z = animation_data[bone_name][anim_step][2];
					const w = animation_data[bone_name][anim_step][3];

					const q = new THREE.Quaternion(x, y, z, w);

					bones[bone_name].rotation.setFromQuaternion(q);
				}
			}

			anim_step += 1;

			if (anim_step > max_anim_step) {
				anim_step = 0;
			}
		}

		// update physics world and threejs renderer
		threeScene.onFrameUpdate();

		counter += 1;

		animation_pointer = requestAnimationFrame(animate);
	}

	onMount(() => {
		// if `animation_path` start with /anim-json-euler-pred
		if (animation_path.startsWith("/anim-json-euler-pred")) {
			frame_skip = 3;
		}

		threeScene = new ThreeScene(
			canvas,
			document.documentElement.clientWidth,
			document.documentElement.clientHeight,
		);

		// -100 is ground level
		threeScene.scene.position.set(0, -1, 0);

		// threeScene.scene.autoUpdate = false;

		Promise.all([loadGLTF(`/glb/dors.glb`), loadJSON(animation_path)]).then(
			([glb_model, anim_data]) => {
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

						if (node.name === "LeftShoulder") {
							// add axes helper
							const axeshelper = new THREE.AxesHelper(1);

							axeshelper.setColors("#f00", "#0f0", "#00f");

							node.add(axeshelper);
						}
					}
				});

				console.log(bones);

				// animation_data = {};

				max_anim_step = anim_data["Hips"].length;

				// animation_data = anim_data;
				// iterate over anim_data object

				animation_data = anim_data;

				console.log(max_anim_step, animation_data);
			},
		);

		animate();
	});

	onDestroy(() => {
		// unsubscribe all stores
		cancelAnimationFrame(animation_pointer);

		threeScene.dispose();
	});
</script>

<section>
	<canvas bind:this={canvas} />
</section>

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
