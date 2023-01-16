<script>

	// imports

	import Cubie from "./Cubie.svelte"
	import { onMount } from "svelte"
	import { fade } from "svelte/transition"
	import { cubieTransform } from "../cubieTransform.js"
	import { TaskQueue } from "../TaskQueue.js"
	import { cubieList } from "../cubieList.js"
	import { faceNames, layerMap } from "../layers.js"
	import { log, mod, sleep, randEl } from "../utils.js"

	let tops = [[3,5,6],[3,5,4],[1,0,5],[7,5,4],[7,3,6],[1,4,3]]

	const ITERATIONS=50 // 50
	// variables

	export let popup
	// export let state
	export let cubeRotation
	export let invHash

	let visible = false
	let scrambling = false
	let transparent = false
	let zoomFactor = 1
	// $: rotationSpeed = scrambling ? 1*100 : 1*250
	$: rotationSpeed = scrambling ? 0 : 0 //5*125
	// let [x,y,z] = [0,0,0] // 3,5,6
	// cubeRotation = { x: x, y: y, z: z }
	let layerTransform = ""

	// let state = 'RBWOGY' // Red Blue White Orange Green Yellow

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

	const hash = {
		BYO : 101, // 200,544,545,645],
		BOW : 103, // 547,546,646],
		BWR : 105, // 541,540,640],
		BRY : 107, // 542,543,206,642,643],

		GOY : 141, // 505,142,242,316],
		GRW : 145, // 146,245,246,356,501],
		GWO : 143, // 507,244,144,336],
		GYR : 147, // 140,240,503,504,604],
		
		OWB : 716, // 25, 26,124,125,224,332,421,521,716,726],
		OGW : 736, // 746,46],
		OYG : 756, // 66,372,756,361,362,160,260,666,766],
		OBY : 776, // 352,776,442,342,706],

		RGY : 316, //732,306,406,42,742],
		RYB : 336, // ,21,712,120,121,326,425,426,524,624],
		RBW : 356, //,772,446,346],
		RWG : 376, //,366,365,265,165,466,752],

		WGR : 314, // 40, 404,730,740],
		WRB : 334, // 126,127,324,424,621,710,721],
		WBO : 354, //  0,344,444,700,770],
		WOG : 374, //  60,161,363,364,566,750,760],

		YOB : 714, // 123,122,321,330,714,724,725],
		YGO : 734, //300,400],
		YRG : 754, // 166,370,754,360,367,460],
		YBR : 774, //  4,350,704,774,340,440],
	}

	invHash = {}
	for (const key in hash) {
		const nr = hash[key]
		invHash[nr] = key
		invHash = invHash
	}
	console.log('Cube',{invHash})

	const rot = (key,lst) => lst.map((i) => key[i]).join('')

	let topIndex = 0
	$: cubeRotation.x = tops[topIndex][0]
	$: cubeRotation.y = tops[topIndex][1]
	$: cubeRotation.z = tops[topIndex][2]

	function rotateCube(direction) { // t ex GWOBYR

		let {x,y,z} = cubeRotation
		let nr = 100*x+10*y+z
		const state = invHash[nr]

		if (direction == "left") {
			if (x==1) cubeRotation.z += 2
			else cubeRotation.y +=2
		}
		if (direction == "right") {
			if (x==1) cubeRotation.z -= 2
			else cubeRotation.y -=2
		}

		if (direction == "up")   topIndex = mod(topIndex+1,6)
		if (direction == "down") topIndex = mod(topIndex-1,6)

		log(topIndex)

		// [x,y,z] = tops[topIndex]
		// cubeRotation = {x,y,z}
		// log(x,y,z,topIndex)
		// x = mod(x,8)
		// y = mod(y,8)
		// z = mod(z,8)
		// const nr = 100*x + 10*y + z
		// state = invHash && invHash[nr] ? invHash[nr] : ""
		// console.log('Cube',{x,y,z,nr,cubeRotation})

	}

	function zoom(factor) {
		zoomFactor *= factor;
	}

	function toggleTransparency() {
		transparent = !transparent;
		console.log({cubeRotation})
		// console.log({state})
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

		b: [addRotation, { layer: "front", orientation: "+" }],
		B: [addRotation, { layer: "front", orientation: "-" }],

		g: [addRotation, { layer: "back", orientation: "-" }],
		G: [addRotation, { layer: "back", orientation: "+" }],

		w: [addRotation, { layer: "top", orientation: "-" }],
		W: [addRotation, { layer: "top", orientation: "+" }],

		y: [addRotation, { layer: "down", orientation: "+" }],
		Y: [addRotation, { layer: "down", orientation: "-" }],

		r: [addRotation, { layer: "left", orientation: "-" }],
		R: [addRotation, { layer: "left", orientation: "+" }],

		o: [addRotation, { layer: "right", orientation: "+" }],
		O: [addRotation, { layer: "right", orientation: "-" }],

		// m: [addRotation, { layer: "middle", orientation: "-" }],
		// M: [addRotation, { layer: "middle", orientation: "+" }],
		// s: [addRotation, { layer: "standing", orientation: "+" }],
		// S: [addRotation, { layer: "standing", orientation: "-" }],
		// e: [addRotation, { layer: "equator", orientation: "+" }],
		// E: [addRotation, { layer: "equator", orientation: "-" }],
		// t: [addRotation, { layer: "top", orientation: "-" }],
		// T: [addRotation, { layer: "top", orientation: "+" }],
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
		// assert("RBWOGY".includes(color),true)
		let command = T1[color]
		assert("LFURBD".includes(command),true)
		if (lower) command = command.toLowerCase()
		return command
	}
	// assert( convert('RBWOGY','R'), 'U')

	function enableKeyControl() {
		document.addEventListener("keydown", (e) => {
			let key = e.key
			console.log(key)
			if ('0123456789'.includes(key)) {
				const sequence = macros[key]
				for (let ch of sequence) {

					// ch = convert(state,ch)

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

	let startX=undefined
	let startY=undefined
	let cx
	let cy
	let ddx=0
	let ddy=0

	// function md(e) {
	// 	startX = e.clientX/5
	// 	startY = e.clientY/5
	// 	cx = cubeRotation.x
	// 	cy = cubeRotation.y
	// 	console.log('md',e.clientX,cubeRotation.x)
	// }

	// function mm(e) {
	// 	if (startX==undefined) return
	// 	const dx = e.clientX/5 - startX
	// 	const dy = e.clientY/5 -startY
	// 	if (Math.abs(ddx-dx)>10) {
	// 		cubeRotation.x = cx+dx
	// 		ddx=dx
	// 	}
	// 	if (Math.abs(ddy-dy)>10) {
	// 		cubeRotation.y = cy+dy
	// 		ddy=dy
	// 	}
	// }

	// function mu(e) {
	// 	startX=undefined
	// 	startY=undefined
	// 	console.log('mu')
	// }

</script>

<!-- <input type="range" id="x" name="rotx" min="0" step=10 max="360" bind:value={cubeRotation.x}>
<label for="rotx">X</label>
<input type="range" id="y" name="roty" min="0" step=10 max="360" bind:value={cubeRotation.y}>
<label for="roty">Y</label>
<input type="range" id="z" name="rotz" min="0" step=10 max="360" bind:value={cubeRotation.z}>
<label for="rotz">Z</label> -->

<!-- on:mousedown={md} on:mousemove={mm} on:mouseup={mu} -->

{#if visible}
	<main in:fade={{ duration: 0 }}  >
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
	input {
		width:300px
	}
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
