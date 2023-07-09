<script lang="ts">
	import WheelPicker, {
		type SelectedEventDetail
	} from '$lib/components/WheelPicker/WheelPicker.svelte';

	let values: string[] = [];
	for (let i = 0; i < 50; i++) {
		values.push('test ' + i);
	}

	let selected = values[0];
	let selectedIndex = 0;

	function onSelected(event: CustomEvent<SelectedEventDetail>) {
		let { value, index } = event.detail;
		selected = value;
		selectedIndex = index;
	}
</script>

<h3>Wheel Picker:</h3>
<div class="wheel-pickers-container">
	<WheelPicker
		on:selected={onSelected}
		{selected}
		width="100px"
		height="250px"
		{values}
		theme="default"
	/>
	<WheelPicker selected="test 25" width="130px" height="150px" {values} theme="default" />
	<WheelPicker selected="test 25" width="85px" height="450px" {values} theme="default" />
	<WheelPicker selected="test 25" width="285px" height="250px" {values} theme="default" />
</div>
<p>First wheel picker has selected: {selected}, with index: {selectedIndex}</p>
<h3>Wheel Picker themes:</h3>
<div class="wheel-pickers-container">
	<WheelPicker width="160px" height="250px" {values} theme="none" />
	<WheelPicker width="160px" height="250px" {values} theme="default" />
	<WheelPicker width="160px" height="250px" {values} theme="inset" />
	<WheelPicker width="160px" height="250px" {values} theme="bard" />
</div>
<h2>TODO:</h2>
<p>selection / events functionality</p>
<p>accessibility</p>

<style lang="scss">
	h3 {
		text-align: center;
	}

	.wheel-pickers-container {
		display: flex;
		justify-content: center;

		> :global(div) {
			margin: 10px;
		}
	}
</style>
