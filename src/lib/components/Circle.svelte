<script lang="ts">
	export let num: number;
	export let size: number;
	export let color: string;
</script>

<div class="rainbow-circle" style="width: {size}px; height: {size}px; background-color: {color};">
	<span class="circle-number" style="font-size: {size * 0.27}px;">{num}</span>
</div>

<style>
	.rainbow-circle {
		position: absolute;
		border-radius: 50%;
		display: flex;
		justify-content: center;
		align-items: center;
		font-weight: bold;
		color: white;
		text-shadow:
			0 0 3px rgba(0, 0, 0, 0.8),
			0 0 5px rgba(0, 0, 0, 0.6);
		box-shadow:
			0 3px 6px rgba(0, 0, 0, 0.16),
			0 3px 6px rgba(0, 0, 0, 0.23);
	}

	.circle-number {
		font-size: 0.8em;
	}
</style>
