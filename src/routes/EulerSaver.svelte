<script>
	/**
	 * play animation at a given frame, used to take screenshot by 'video-recorder'
	 */
	import _ from "lodash";
	import axios from "axios";
	import { onDestroy, onMount } from "svelte";
	import * as THREE from "three";
	import ThreeScene from "../lib/ThreeScene";
	import { loadFBX, loadGLTF, loadJSON } from "../utils/ropes";

	/** @type {HTMLCanvasElement} */
	let canvas;

	/** @type {ThreeScene} */
	let threeScene;

	/** @type {THREE.AnimationMixer} */
	let anim_mixer;
	/** @type {THREE.AnimationAction} */
	let anim_action;

	let animation_pointer = 0;

	const clock = new THREE.Clock();

	// model file name
	export let model;

	function get_longest_track(tracks) {
		let max_len = 0;
		let max_times = [];

		for (const v of tracks) {
			if (v.times.length > max_len) {
				max_len = v.times.length;
				max_times = v.times;
			}
		}

		return max_times;
	}

	onMount(() => {
		threeScene = new ThreeScene(
			canvas,
			document.documentElement.clientWidth,
			document.documentElement.clientHeight,
		);

		// -100 is ground level
		threeScene.scene.position.set(0, -150, 0);

		// check if model has ".fbx" in it
		let model_task = null;
		let model_type = "";

		if (model.indexOf(".fbx") !== -1) {
			model_task = loadFBX(`/fbx/${model}`);
			model_type = "fbx";
		} else if (model.indexOf(".glb") !== -1) {
			model_task = loadGLTF(`/glb/${model}`);
			model_type = "glb";
		}

		if (model_task === null) {
			throw new Error("Model file type not supported");
		}

		Promise.all([model_task, loadJSON(`/filenames.json`)]).then(
			([model_mesh, filenames]) => {
				if (model_type === "glb") {
					model_mesh = model_mesh.scene.children[0];
				}

				model_mesh.name = "diva";

				anim_mixer = new THREE.AnimationMixer(model_mesh);
				// console.log(fbx_unity_anim);
				threeScene.scene.add(model_mesh);

				return;

				(async () => {
					const start_idx = filenames.indexOf("Step Backward.json");

					for (let i = start_idx; i < filenames.length; i++) {
						const anim_data = await loadJSON(
							`/anim-json/${filenames[i]}`,
						);

						const clip = THREE.AnimationClip.parse(anim_data);

						anim_action = anim_mixer.clipAction(clip);
						anim_action.reset();
						anim_action.setLoop(THREE.LoopRepeat, 1);
						// keep model at the position where it stops
						anim_action.clampWhenFinished = true;
						anim_action.enabled = true;
						// anim_action.fadeIn(0.5);

						anim_action.paused = true;

						const max_times = get_longest_track(clip.tracks);

						const euler_data = {};

						model_mesh.traverse((node) => {
							// @ts-ignore
							if (node.isBone) {
								// @ts-ignore
								if (euler_data[node.name] === undefined) {
									// somehow maximo has double bones, so only use the first one
									euler_data[node.name] = [];
								}
							}
						});

						// console.log(euler_data);

						for (let j = 0; j < max_times.length; j++) {
							anim_action.time = max_times[j];

							anim_action.play();

							anim_mixer.update(clock.getDelta());

							threeScene.onFrameUpdate();

							// get the euler angles of each joints
							model_mesh.traverse((node) => {
								// @ts-ignore
								if (node.isBone) {
									euler_data[node.name].push([
										node.rotation.x,
										node.rotation.y,
										node.rotation.z,
									]);
								}
							});

							// await new Promise((resolve) => {
							// 	setTimeout(() => {
							// 		resolve();
							// 	}, 100);
							// });
						}

						console.log(euler_data);

						// todo, save `euler_data` to local file
						const headers = {
							"Content-Type": "application/json",
						};

						const reponse = await axios.post(
							"http://localhost:2020",
							{
								data: euler_data,
								name: filenames[i].replace(".json", ""),
							},
							headers,
						);

						console.log(reponse.data.message);

						// break;
					}
				})();
			},
		);

		// animate();
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

<div class="cover"></div>

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

	.cover {
		position: absolute;
		top: 0;
		left: 0;
		bottom: 0;
		right: 0;
		background-color: rgba(0, 0, 0, 1);
	}
</style>
