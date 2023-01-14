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




	// const hash = {}
	// 						// lr ud pg

	// hash.BYOGVR =  "xxx"
	// hash.BOVGRY =  "xxx"
	// hash.BVRGYO =  "xxx"
	// hash.BRYGOV =  "xxx"

	// hash.GRVBOY =  "xzx"
	// hash.GYRBVO =  "xxx"
	// hash.GOYBRV =  "xxx"
	// hash.GVOBYR =  "xxx"

	// hash.OGVRBY =  "xxx"
	// hash.OVBRYG =  "xxx"
	// hash.OBYRGV =  "xzy"
	// hash.OYGRVB =  "xxx"

	// hash.RGYOBV =  "xxx"
	// hash.RYBOVG =  "xxx"
	// hash.RBVOGY =  "xzy"
	// hash.RVGOYB =  "xxx"

	// hash.VGRYBO =  "xxx"
	// hash.VRBYOG =  "xxx"
	// hash.VBOYGR =  "xzy"
	// hash.VOGYRB =  "xxx"

	// hash.YGOVBR =  "xxx"
	// hash.YOBVRG =  "xxx"
	// hash.YBRVGO =  "xxy"
	// hash.YRGVOB =  "xxx"

	const rot = (key,lst) => lst.map((i) => key[i]).join('')

	function rotateCube(direction) { // t ex GVOBYR

		console.log(state)

		// let [lr,ud,pg] = hash[state]
		// console.log({state,lr,ud,pg})
		// console.log(hash[state])
		let pg
		let lr
		let ud

		if ('BG'.includes(state[0])) pg='x'
		if ('YV'.includes(state[0])) pg='y'
		if ('OR'.includes(state[0])) pg='z'

		if ('BG'.includes(state[1])) lr='x'
		if ('YV'.includes(state[1])) lr='y'
		if ('OR'.includes(state[1])) lr='z'

		if ('BG'.includes(state[2])) ud='x'
		if ('YV'.includes(state[2])) ud='y'
		if ('OR'.includes(state[2])) ud='z'

		console.log({pg,lr,ud})


		if (direction == "left")  cubeRotation[lr] += 2
		if (direction == "right") cubeRotation[lr] -= 2

		if (direction == "up")    cubeRotation[ud] += 2
		if (direction == "down")  cubeRotation[ud] -= 2

		if (direction == "pu")    cubeRotation[pg] += 2
		if (direction == "pd")    cubeRotation[pg] -= 2

		let vector

		if (direction == "left") {
			if (lr=='x') vector = [0,5,1,3,2,4]
			if (lr=='y') vector = [5,1,0,2,4,3]
			if (lr=='z') vector = [1,3,2,4,0,5]
		} 
		if (direction == "right") vector = [0,5,1,3,2,4] // 0 3
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
			style:--rotation-x="{cubeRotation.x*45}deg"
			style:--rotation-y="{cubeRotation.y*45}deg"
			style:--rotation-z="{cubeRotation.z*45}deg"
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
