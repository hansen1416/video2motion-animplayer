<script>
	import { Router, Route } from "svelte-routing";
	import Home from "./routes/Home.svelte";
	import Loader from "./routes/Loader.svelte";
	import Camera from "./routes/Camera.svelte";
	import Glb from "./routes/Glb.svelte";
	import GlbEuler from "./routes/GlbEuler.svelte";
	import GlbQuaternion from "./routes/GlbQuaternion.svelte";
	import EulerSaver from "./routes/EulerSaver.svelte";
	import EulerVisual from "./routes/EulerVisual.svelte";
	import PlayLocal from "./routes/PlayLocal.svelte";
</script>

<Router>
	<div>
		<Route path="/:model/:anim/:elevation/:azimuth/:step" let:params>
			<Home
				model={params.model}
				anim={params.anim}
				step={params.step}
				elevation={params.elevation}
				azimuth={params.azimuth}
			/>
		</Route>
		<Route path="/" component={Loader} />
		<Route path="/glb" component={Glb} />
		<Route path="/glb-euler" component={GlbEuler} />
		<Route path="/glb-quat" component={GlbQuaternion} />
		<Route path="/camera" component={Camera} />
		<Route path="/:anim/euler-saver" component={EulerSaver} />
		<Route path="/euler-visual" component={EulerVisual} />
		<Route path="/play-local" component={PlayLocal} />
	</div>
</Router>
