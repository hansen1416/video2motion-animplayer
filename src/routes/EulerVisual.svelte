<script>
	/**
	 * visualizes the euler angles from the server
	 */
	import _ from "lodash";
	import axios from "axios";
	import { onDestroy, onMount } from "svelte";
	import ThreeScene from "../lib/ThreeScene";
	import { loadGLTF } from "../utils/ropes";

	/** @type {HTMLCanvasElement} */
	let canvas;

	/** @type {ThreeScene} */
	let threeScene;

	let animation_pointer = 0;

	let bones = {};
	let animation_data = null;

	let anim_step = 0;
	let max_anim_step = 29;

	let counter = 0;
	let direction = 1;

	let skip_frames = 3;

	const bones_to_use = [
		"Hips",
		"RightUpLeg",
		"RightLeg",
		"LeftUpLeg",
		"LeftLeg",
		"Spine",
		"Spine1",
		"Spine2",
		"Neck",
		"Head",
		"RightShoulder",
		"RightArm",
		"RightForeArm",
		"LeftShoulder",
		"LeftArm",
		"LeftForeArm",
	];

	function animate() {
		if (animation_data && bones && counter % skip_frames === 0) {
			for (let i in animation_data[anim_step]) {
				const bone_name = bones_to_use[i];

				bones[bone_name].rotation.x = animation_data[anim_step][i][0];
				bones[bone_name].rotation.y = animation_data[anim_step][i][1];
				bones[bone_name].rotation.z = animation_data[anim_step][i][2];
			}

			// console.log(anim_step, animation_data[anim_step]);

			if (direction > 0) {
				anim_step += 1;
			} else {
				anim_step -= 1;
			}

			if (anim_step >= max_anim_step || anim_step <= 0) {
				direction *= -1;
			}
		}

		// update physics world and threejs renderer
		threeScene.onFrameUpdate();

		counter += 1;

		animation_pointer = requestAnimationFrame(animate);
	}

	onMount(() => {
		// if `animation_path` start with /anim-json-euler-pred

		threeScene = new ThreeScene(
			canvas,
			document.documentElement.clientWidth,
			document.documentElement.clientHeight,
		);

		// -100 is ground level
		threeScene.scene.position.set(0, -1, 0);

		const headers = {
			"Content-Type": "application/json",
		};

		Promise.all([
			loadGLTF(`/glb/dors.glb`),
			axios.get(
				"http://localhost:5000/",
				{ params: { idx: 9 } },
				headers,
			),
		]).then(([glb_model, euler_data]) => {
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

			animation_data = euler_data.data.data;
		});

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
