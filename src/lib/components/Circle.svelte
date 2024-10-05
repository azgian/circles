<script lang="ts">
	export let num: number;
	export let size: number;
	export let color: string;
</script>

<div class="circle" style="width: {size}px; height: {size}px; background-color: {color};">
	<span class="number">{num}</span>
</div>

<style>
	.circle {
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: 50%;
		font-weight: bold;
		font-size: 1.5em;
		color: white;
		user-select: none;
	}

	.number {
		text-shadow:
			-1px -1px 0 #000,
			1px -1px 0 #000,
			-1px 1px 0 #000,
			1px 1px 0 #000,
			2px 2px 4px rgba(0, 0, 0, 0.5);
	}
</style>
