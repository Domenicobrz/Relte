<script lang="ts">
	import WheelPicker from '$lib/components/WheelPicker/WheelPicker.svelte';

	let values: string[] = [];
	for (let i = 0; i < 50; i++) {
		values.push('test ' + i);
	}
</script>

<h3>Wheel Picker:</h3>
<div class="wheel-pickers-container">
	<WheelPicker width="100px" height="250px" {values} theme="default" />
	<WheelPicker width="130px" height="150px" {values} theme="default" />
	<WheelPicker width="85px" height="450px" {values} theme="default" />
	<WheelPicker width="285px" height="250px" {values} theme="default" />
</div>
<h3>Wheel Picker themes:</h3>
<div class="wheel-pickers-container">
	<WheelPicker width="160px" height="250px" {values} theme="none" />
	<WheelPicker width="160px" height="250px" {values} theme="default" />
	<WheelPicker width="160px" height="250px" {values} theme="inset" />
	<WheelPicker width="160px" height="250px" {values} theme="bard" />
</div>
<h2>TODO: styling presets with css / scss preprocessors:</h2>
<pre>
	style lang="scss"
		@import './styles/colors.scss';
		@import './styles/typography.scss';
		/* Etc. */
	style
</pre>
<p>
	"when using preprocessors like `svelte-preprocess` in Svelte, your styles still remain scoped to
	the specific component, even when you're importing styles from external files."
</p>
<p>
	Check if this is actually true and if there's a way for the user to specify it's own imports
	(maybe ask ChatGPT)
</p>

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
