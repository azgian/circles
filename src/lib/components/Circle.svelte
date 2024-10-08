<script lang="ts">
	export let num: number;
	export let size: number;
	export let color: string;
	export let isPulsing: boolean = false;
</script>

<div
	class="circle"
	style="width: {size}px; height: {size}px; background-color: {color};"
	class:pulsing={isPulsing}
>
	<span class="number" class:pulsing={isPulsing}>{num}</span>
</div>

<style>
	.circle {
		border-radius: 50%;
		display: flex;
		justify-content: center;
		align-items: center;
		font-weight: bold;
		color: white;
		user-select: none;
	}

	.number {
		font-size: 20px;
		text-shadow:
			-1px -1px 0 #000,
			1px -1px 0 #000,
			-1px 1px 0 #000,
			1px 1px 0 #000,
			0px 2px 2px rgba(0, 0, 0, 0.5);
	}

	.pulsing {
		animation: pulse 0.3s 3;
	}

	@keyframes pulse {
		0%,
		100% {
			transform: scale(1);
		}
		50% {
			transform: scale(1.2);
		}
	}
</style>
