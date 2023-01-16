<script>
	import Cube from "./components/Cube.svelte";
	import Help from "./components/Help.svelte";
	import Popup from "./components/Popup.svelte";
	let popup = { show: false, text: "" };
	// let state = 'RBW'
	let cubeRotation = {x:3,y:5,z:6} // 3 5 6
	let invHash = {}
</script>

<Cube bind:popup bind:cubeRotation bind:invHash/>
<Help bind:cubeRotation bind:invHash/>
<Popup bind:popup />

<style>
	:global(*) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:global(body) {
		background-color: black;
		color: white;
	}
</style>
