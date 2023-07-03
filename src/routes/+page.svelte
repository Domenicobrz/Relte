<script lang="ts">
	import WheelPicker from '$lib/components/WheelPicker.svelte';

	let values: string[] = [];
	for (let i = 0; i < 50; i++) {
		values.push('test ' + i);
	}
</script>

<p>Components:</p>
<div class="wheel-pickers-container">
	<WheelPicker width="100px" height="250px" {values} />
	<WheelPicker width="130px" height="150px" {values} />
	<WheelPicker width="85px" height="450px" {values} />
</div>

<style lang="scss">
	.wheel-pickers-container {
		display: flex;

		> :global(div) {
			margin: 10px;
		}
	}
</style>
