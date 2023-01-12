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
	let visible = false
	let scrambling = false
	let transparent = false
	let zoomFactor = 1
	// $: rotationSpeed = scrambling ? 1*100 : 1*250
	$: rotationSpeed = scrambling ? 50 : 125
	const [x,y,z] = [3,5,6]
	const cubeRotation = { x: 45*x, y: 45*y, z: 45*z }
	let layerTransform = ""

	let state = 'RBVOGY' // Röd Blå Vit Orange Grön Yellow

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

	hash.OGVRBY = 312 // <=> x=3*45 y=1*45 z=2*45 grader
	hash.OVBRYG = 332
	hash.OBYRGV = 352
	hash.OYGRVB = 372

	hash.RGYOBV = 316
	hash.RYBOVG = 336
	hash.RBVOGY = 356
	hash.RVGOYB = 376

	hash.VGRYBO = 314
	hash.VRBYOG = 334
	hash.VBOYGR = 354
	hash.VOGYRB = 374

	hash.YGOVBR = 310
	hash.YOBVRG = 330
	hash.YBRVGO = 350
	hash.YRGVOB = 370

	hash.BYOGVR = 101
	hash.BOVGRY = 103
	hash.BVRGYO = 105
	hash.BRYGOV = 107

	hash.GOYBRV = 141
	hash.GVOBYR = 143
	hash.GRVBOY = 145
	hash.GYRBVO = 147

	const rot = (key,lst) => lst.map((i) => key[i]).join('')

	function save(state,lst) {
		console.log('save in',state,lst)
		state = rot(state,lst)
		console.log('save ut',state)
		return state
	}

	function rotateCube(direction) { // t ex GVOBYR
		console.log(direction)
		if (direction == "up")    state = save(state,[0,2,4,3,5,1])
		if (direction == "down")  state = save(state,[0,5,1,3,2,4])
		if (direction == "left")  state = save(state,[2,1,3,5,4,0])
		if (direction == "right") state = save(state,[5,1,0,2,4,3])
		if (direction == "pu")    state = save(state,[1,3,2,4,0,5])
		if (direction == "pd")    state = save(state,[4,0,2,1,3,5])

		let xyz = hash[state]
		console.log({state,xyz})
		let z = Math.round(xyz % 10)
		xyz = Math.floor(xyz / 10)
		let y = Math.round(xyz % 10)
		xyz = Math.floor(xyz / 10)
		let x = Math.round(xyz % 10)

		console.log({x,y,z})

		if (x-cubeRotation.x > 4) x=x-8
		if (y-cubeRotation.y > 4) y=y-8
		if (z-cubeRotation.z > 4) z=z-8

		if (x-cubeRotation.x < -4) x=x+8
		if (y-cubeRotation.y < -4) y=y+8
		if (z-cubeRotation.z < -4) z=z+8

		cubeRotation.x = 45*x
		cubeRotation.y = 45*y
		cubeRotation.z = 45*z

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
		u: [undoRotation],
		U: [resetCube],
		X: [scrambleCube],
		x: [stopScrambling],
		f: [addRotation, { layer: "front", orientation: "+" }],
		F: [addRotation, { layer: "front", orientation: "-" }],
		b: [addRotation, { layer: "back", orientation: "-" }],
		B: [addRotation, { layer: "back", orientation: "+" }],
		t: [addRotation, { layer: "top", orientation: "-" }],
		T: [addRotation, { layer: "top", orientation: "+" }],
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
		'3': 'rtRT',     // C 6 (after 6 macros we reach starting position again)
		'4': 'trTRTFtf', // C 15
		'5': 'TLtltfTF', // C 15
		'6': 'frtRTF',   // C 6
		'7': 'trTLtRTl', // C 3
		'8': 'tRTr'      // C 6
	}

	const convert = (state,ch) => {
		if (! 'TFRDBLtfrdbl'.includes(ch)) return ch
		const T0 = 'TFRDBL'
		const T1 = {
			 'R': 'L',
			 'B': 'F',
			 'V': 'T',
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
		assert("LFTRBD".includes(command),true)
		if (lower) command = command.toLowerCase()
		return command
	}
	assert( convert('RBVOGY','R'), 'T')

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
		transform: scale(var(--zoom)) rotateX(var(--rotation-x))
			rotateY(var(--rotation-y)) rotateZ(var(--rotation-z));
		transition: transform var(--speed) ease-out;
	}

	.rotationLayer {
		transition: transform var(--speed) ease-out;
	}
</style>
