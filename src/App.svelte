<script>
	import Cube from "./components/Cube.svelte";
	import Help from "./components/Help.svelte";
	import Popup from "./components/Popup.svelte";
	let popup = { show: false, text: "" };
	let state = 'RBVOGY'
	let cubeRotation = {x:3*45,y:5*45,z:6*45} // 3 5 6
</script>

<Help bind:state bind:cubeRotation/>
<Cube bind:popup bind:state bind:cubeRotation/>
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
