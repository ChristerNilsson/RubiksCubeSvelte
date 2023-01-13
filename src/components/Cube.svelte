<script>

	// imports

	import Cubie from "./Cubie.svelte"
	import { onMount } from "svelte"
	import { fade } from "svelte/transition"
	import { cubieTransform } from "../cubieTransform.js"
	import { TaskQueue } from "../TaskQueue.js"
	import { cubieList } from "../cubieList.js"
	import { faceNames, layerMap } from "../layers.js"
	import { mod, sleep, randEl } from "../utils.js"

	const ITERATIONS=50 // 50
	// variables

	export let popup
	export let state
	export let cubeRotation

	let visible = false
	let scrambling = false
	let transparent = false
	let zoomFactor = 1
	// $: rotationSpeed = scrambling ? 1*100 : 1*250
	$: rotationSpeed = scrambling ? 5*50 : 5*125
	let [x,y,z] = [0,0,0] // 3,5,6
	cubeRotation = { x: 45*x, y: 45*y, z: 45*z }
	let layerTransform = ""

	// let state = 'RBVOGY' // Röd Blå Vit Orange Grön Yellow

	// cubies

	let cubies = cubieList.map(function (cubie) {
		return {
			...cubie,
			rotating: false,
			originalCoords: { ...cubie.coords },
			originalColors: { ...cubie.colors },
		}
	})

	function assert(a,b) {
		if (a!=b) {
			console.log('Assert failed:')
			console.log('  a:',a)
			console.log('  b:',b)
			console.assert(a==b)
		}
	}

	// various cube functions

	function checkSolved() {
		const cubeIsSolved = faceNames.every(
			(faceName) =>
				new Set(cubies.map((cubie) => cubie.colors[faceName]))
					.size == 2
		);
		if (cubeIsSolved) {
			showPopup("Cube is solved! 🎉")
		}
	}




	const hash = {}

	hash.BYOGVR =  {x:1, y:0, z:1}
	hash.BOVGRY =  {x:1, y:0, z:3}
	hash.BVRGYO =  {x:1, y:0, z:5}
	hash.BRYGOV =  {x:1, y:0, z:7}

	hash.GRVBOY =  {x:5, y:0, z:1}
	hash.GYRBVO =  {x:5, y:0, z:3}
	hash.GOYBRV =  {x:5, y:0, z:5}
	hash.GVOBYR =  {x:5, y:0, z:7}

	hash.OGVRBY =  {x:3, y:1, z:2}
	hash.OVBRYG =  {x:3, y:3, z:2}
	hash.OBYRGV =  {x:3, y:5, z:2}
	hash.OYGRVB =  {x:3, y:7, z:2}

	hash.RGYOBV =  {x:3, y:1, z:6}
	hash.RYBOVG =  {x:3, y:3, z:6}
	hash.RBVOGY =  {x:3, y:5, z:6}
	hash.RVGOYB =  {x:3, y:7, z:6}

	hash.VGRYBO =  {x:3, y:1, z:4}
	hash.VRBYOG =  {x:3, y:3, z:4}
	hash.VBOYGR =  {x:3, y:5, z:4}
	hash.VOGYRB =  {x:3, y:7, z:4}

	hash.YGOVBR =  {x:3, y:1, z:0}
	hash.YOBVRG =  {x:3, y:3, z:0}
	hash.YBRVGO =  {x:3, y:5, z:0}
	hash.YRGVOB =  {x:3, y:7, z:0}

	const rot = (key,lst) => lst.map((i) => key[i]).join('')

	function rotateCube(direction) { // t ex GVOBYR

		console.log(state)

		let {x,y,z} = hash[state]
		x *= 45
		y *= 45
		z *= 45
		cubeRotation = {x,y,z}

		let vector

		if (direction == "left")  vector = [0,5,1,3,2,4] // 0 3
		if (direction == "right") vector = [0,2,4,3,5,1] // 0 3

		if (direction == "pu")    vector = [5,1,0,2,4,3] // 1 4
		if (direction == "pd")    vector = [2,1,3,5,4,0] // 1 4

		if (direction == "down")  vector = [1,3,2,4,0,5] // 2 5
		if (direction == "up")    vector = [4,0,2,1,3,5] // 2 5

		state = rot(state,vector)
	
	}

	function zoom(factor) {
		zoomFactor *= factor;
	}

	function toggleTransparency() {
		transparent = !transparent;
		console.log({cubeRotation})
		console.log({state})
	}

	async function rotateLayer({ layer, orientation }) {
		console.log('rotateLayer',layer, orientation)
		const { axis, value } = layerMap[layer];
		const trafo = cubieTransform[layer][orientation];
		for (let index = 0; index < cubies.length; index++) {
			cubies[index].rotating =
				cubies[index].coords[axis] == value;
		}
		layerTransform = `rotate${axis.toUpperCase()}(${orientation}90deg)`;
		await sleep(rotationSpeed);
		for (let index = 0; index < cubies.length; index++) {
			if (cubies[index].rotating) {
				cubies[index] = trafo(cubies[index]);
				cubies[index].rotating = false;
			}
		}
		layerTransform = "";
		await sleep(0);
		checkSolved();
	}

	function showPopup(message, duration = 2000) {
		popup = { show: true, text: message };
		setTimeout(() => {
			popup.show = false;
		}, duration);
	}

	function resetCube() {
		if (rotationQueue.executing || scrambling) return;
		rotationQueue.clearHistory();
		for (let index = 0; index < cubies.length; index++) {
			cubies[index].coords = {
				...cubies[index].originalCoords,
			};
			cubies[index].colors = {
				...cubies[index].originalColors,
			};
		}
	}

	async function scrambleCube(iterations = ITERATIONS) {
		if (rotationQueue.executing || scrambling) return;
		scrambling = true;
		rotationQueue.clearHistory();
		for (let i = 0; i < iterations; i++) {
			await rotateLayer({
				layer: randEl(Object.keys(layerMap)),
				orientation: randEl(["+", "-"]),
			});
			await sleep(20);
			if (!scrambling) break;
		}

		scrambling = false;
	}

	function stopScrambling() {
		scrambling = false;
		rotationQueue.stop();
	}

	// rotation queue

	function invertRotation({ layer, orientation }) {
		return {
			layer,
			orientation: orientation == "+" ? "-" : "+",
		};
	}

	const rotationQueue = new TaskQueue(rotateLayer, invertRotation);

	function addRotation(data) {
		if (scrambling) return;
		rotationQueue.add(data);
	}

	function undoRotation() {
		rotationQueue.undo();
	}

	// key controller

	const keyController = {
		ArrowLeft:  [rotateCube, "left"],
		ArrowRight: [rotateCube, "right"],
		ArrowUp:    [rotateCube, "up"],
		ArrowDown:  [rotateCube, "down"],
		PageUp:     [rotateCube, "pu"],
		PageDown:   [rotateCube, "pd"],
		"+": [zoom, 1.15],
		"-": [zoom, 1 / 1.15],
		c: [toggleTransparency],
		z: [undoRotation],
		Z: [resetCube],
		X: [scrambleCube],
		x: [stopScrambling],
		f: [addRotation, { layer: "front", orientation: "+" }],
		F: [addRotation, { layer: "front", orientation: "-" }],
		b: [addRotation, { layer: "back", orientation: "-" }],
		B: [addRotation, { layer: "back", orientation: "+" }],
		// t: [addRotation, { layer: "top", orientation: "-" }],
		// T: [addRotation, { layer: "top", orientation: "+" }],
		u: [addRotation, { layer: "top", orientation: "-" }],
		U: [addRotation, { layer: "top", orientation: "+" }],
		d: [addRotation, { layer: "down", orientation: "+" }],
		D: [addRotation, { layer: "down", orientation: "-" }],
		l: [addRotation, { layer: "left", orientation: "-" }],
		L: [addRotation, { layer: "left", orientation: "+" }],
		r: [addRotation, { layer: "right", orientation: "+" }],
		R: [addRotation, { layer: "right", orientation: "-" }],
		// m: [addRotation, { layer: "middle", orientation: "-" }],
		// M: [addRotation, { layer: "middle", orientation: "+" }],
		// s: [addRotation, { layer: "standing", orientation: "+" }],
		// S: [addRotation, { layer: "standing", orientation: "-" }],
		// e: [addRotation, { layer: "equator", orientation: "+" }],
		// E: [addRotation, { layer: "equator", orientation: "-" }],
	};

	const macros = {
		'3': 'ruRU',     // C 6 (after 6 macros we reach starting position again)
		'4': 'urURUFuf', // C 15
		'5': 'ULulufUF', // C 15
		'6': 'fruRUF',   // C 6
		'7': 'urULuRUl', // C 3
		'8': 'uRUr'      // C 6
	}

	const convert = (state,ch) => {
		if (! 'UFRDBLufrdbl'.includes(ch)) return ch
		const T0 = 'UFRDBL'
		const T1 = {
			 'R': 'L',
			 'B': 'F',
			 'V': 'U',
			 'O': 'R',
			 'G': 'B',
			 'Y': 'D'
		}

		const lower = ch.toLowerCase() == ch
		ch = ch.toUpperCase()
		const index = T0.indexOf(ch)
		console.log(ch,index)
		assert([0,1,2,3,4,5].includes(index),true)
		const color = state[index]
		assert("RBVOGY".includes(color),true)
		let command = T1[color]
		assert("LFURBD".includes(command),true)
		if (lower) command = command.toLowerCase()
		return command
	}
	assert( convert('RBVOGY','R'), 'U')

	function enableKeyControl() {
		document.addEventListener("keydown", (e) => {
			let key = e.key
			console.log(key)
			if ('0123456789'.includes(key)) {
				const sequence = macros[key]
				for (let ch of sequence) {

					ch = convert(state,ch)

					console.log(ch)
					const [fun, arg] = keyController[ch]
					fun(arg)
				}
			} else {
				if (Object.keys(keyController).includes(key)) {

					if (key.length==1) key = convert(state,key)
					
					const [fun, arg] = keyController[key]
					fun(arg)
				}
			}
		});
	}

	onMount(() => {
		visible = true;
		enableKeyControl();
	});
</script>

{#if visible}
	<main in:fade={{ duration: 10 }}>
		<div
			class="cube"
			style:--speed="{rotationSpeed}ms"
			style:--zoom={zoomFactor}
			style:--rotation-x="{cubeRotation.x}deg"
			style:--rotation-y="{cubeRotation.y}deg"
			style:--rotation-z="{cubeRotation.z}deg"
		>
			<div class="cubieContainer">
				{#each cubies.filter((c) => !c.rotating) as cubie (cubie.id)}
					<Cubie {cubie} {transparent} />
				{/each}
			</div>
			<div
				class="rotationLayer"
				style:--speed="{layerTransform ? rotationSpeed : 0}ms"
				style:transform={layerTransform}
			>
				{#each cubies.filter((c) => c.rotating) as cubie (cubie.id)}
					<Cubie {cubie} {transparent} />
				{/each}
			</div>
		</div>
	</main>
{/if}

<style>
	main {
		--cubie-size: min(120px, 18vw);
		min-height: 100vh;
		perspective: calc(10 * var(--cubie-size));
		transform-style: preserve-3d;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	:global(main *) {
		transform-style: inherit;
		position: absolute;
	}

	.cube {
		transform: 
			scale(var(--zoom))
			rotateX(var(--rotation-x))
			rotateY(var(--rotation-y))
			rotateZ(var(--rotation-z));
		transition: transform var(--speed) ease-out;
	}

	.rotationLayer {
		transition: transform var(--speed) ease-out;
	}
</style>
