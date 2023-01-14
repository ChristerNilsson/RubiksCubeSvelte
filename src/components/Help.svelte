<script>
	import { fly } from "svelte/transition"

	export let state
	export let cubeRotation

	let showHelp = false

	let helpShown = localStorage.getItem("helpShown")

	if (helpShown != "yes") {
		setTimeout(() => {
			showHelp = true
			localStorage.setItem("helpShown", "yes")
		}, 1000)
	}

	document.addEventListener("keydown", (e) => {
		if (e.key == "i") {
			showHelp = !showHelp;
		}
	})
</script>

<section>
	<input type="checkbox" id="helpToggler" bind:checked={showHelp} />
	<label for="helpToggler"><i class="fas fa-info-circle" /></label>
	{state} {cubeRotation.x} {cubeRotation.y} {cubeRotation.z}
	{#if showHelp}
		<div transition:fly={{ x: -150 }} id="helpContent">
			<div class="for-mobile">
				<p>
					<strong>
						It seems that you are on a mobile device. You
						need a keyboard to control the cube.
					</strong>
				</p>
			</div>
			<div class="for-desktop">
				<p>The key <strong>i</strong> opens/closes this menu.</p>
				<br />
				<p>The keys <strong>Left</strong>/<strong>Right</strong>, <strong>Up</strong>/<strong>Down</strong> and <strong>PgUp</strong>/<strong>PgDn</strong> rotates the cube.</p>
				<p>The keys <strong>+</strong>/<strong>-</strong> zoom the cube in/out.</p>
				<br />
				<p>Direction: <strong>clock-wise</strong>/<strong>Anti Clock-Wise</strong>.</p>
				<p>The keys <strong>u</strong>/<strong>U</strong> rotate the upper layer.</p>
				<p>The keys <strong>d</strong>/<strong>D</strong> rotate the down layer.</p>
				<p>The keys <strong>l</strong>/<strong>L</strong> rotate the left layer.</p>
				<p>The keys <strong>r</strong>/<strong>R</strong> rotate the right layer.</p>
				<p>The keys <strong>f</strong>/<strong>F</strong> rotate the front layer.</p>
				<p>The keys <strong>b</strong>/<strong>B</strong> rotate the back layer.</p>

				<!-- <p>The keys <strong>e</strong>,<strong>E</strong> rotate the equator layer.</p>
				<p>The keys <strong>m</strong>,<strong>M</strong> rotate the middle layer.</p>
				<p>The keys <strong>s</strong>,<strong>S</strong> rotate the standing layer.</p> -->

				<p>When you enter many rotations in a row,<br />they are put on a queue.</p>
				<br />
				<p>The key <strong>3</strong> performs r u R U</p>
				<p>The key <strong>4</strong> performs u r U R U F u f</p>
				<p>The key <strong>5</strong> performs U L u l u f U F</p>
				<p>The key <strong>6</strong> performs f r u R U F</p>
				<p>The key <strong>7</strong> performs u r U L u R U l </p>
				<p>The key <strong>8</strong> performs u R U r</p>
				<br />
				<p>The key <strong>z</strong> undoes the last layer rotation.</p>
				<p>The key <strong>Z</strong> resets the whole cube.</p>
				<p>The key <strong>x</strong> stops the rotation queue.</p>
				<p>The key <strong>X</strong> scrambles the cube.</p>
				<p>The key <strong>c</strong> toggles transparent mode.</p>
			</div>
			<br />
			<p>
				<a
					href="https://github.com/ScriptRaccoon/RubiksCubeSvelte"
					target="_blank">GitHub repository</a>
			</p>
		</div>
	{/if}
</section>

<style>
	section {
		padding: 20px 0px 0px 20px;
		font-family: "Courier New", Courier, monospace;
		font-size: 22px;
		position: absolute;
		inset: 0 auto 0 0;
		z-index: 1;
	}

	#helpToggler {
		display: none;
	}

	#helpContent {
		background-color: rgba(0, 0, 0, 0.9);
		box-shadow: 0px 0px 20px black;
		flex-grow: 1;
		padding: 0px 20px 20px 0px;
	}

	a {
		color: yellow;
	}
	a:hover {
		color: orange;
	}

	i {
		opacity: 0.6;
		cursor: pointer;
		text-shadow: 0px 0px 20px #fff8;
	}

	i:hover {
		opacity: 1;
	}

	#helpContent p {
		margin-top: 10px;
	}

	#helpContent strong {
		color: skyblue;
	}

	.for-mobile {
		display: none;
	}

	@media (pointer: none), (pointer: coarse) {
		.for-mobile {
			display: block;
		}
		.for-desktop {
			display: none;
		}
	}
</style>
