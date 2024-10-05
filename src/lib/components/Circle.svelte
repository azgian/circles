<script lang="ts">
	export let num: number = 0;
	export let size: number;
	export let color: string;
</script>

<div class="circle" style="width: {size}px; height: {size}px; background-color: {color};">
	{#if num !== 0}
		<span class="circle-number" style="font-size: {size * 0.27}px;">{num}</span>
	{/if}
</div>

<style>
	.circle {
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: 50%;
	}

	.circle-number {
		color: white;
		font-weight: bold;
	}
</style>
