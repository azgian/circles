<script lang="ts">
	import { onMount } from 'svelte';
	import '../app.css';

	let spotlight: HTMLDivElement;

	const animateSpotlight = () => {
		let x = window.innerWidth - 100;
		let y = 0;
		let dx = 0.95;
		let dy = 0.75;

		const animate = () => {
			if (x > window.innerWidth || x < 400) dx = -dx;
			if (y > 300 || y < -100) dy = -dy;
			x += dx;
			y += dy;
			if (spotlight) {
				spotlight.style.transform = `translate(${x}px, ${y}px)`;
			}
			requestAnimationFrame(animate);
		};

		animate();
	};

	onMount(() => {
		animateSpotlight();
	});
</script>

<div class="spotlight" bind:this={spotlight}></div>

<slot />

<style>
	.spotlight {
		position: fixed;
		top: -100px;
		left: -100px;
		width: 400px;
		height: 400px;
		background: radial-gradient(
			circle,
			rgba(255, 165, 0, 0.6) 0%,
			rgba(255, 140, 0, 0.3) 30%,
			rgba(255, 69, 0, 0) 70%
		);
		pointer-events: none;
		opacity: 0.8;
		filter: blur(0px);
	}
</style>
