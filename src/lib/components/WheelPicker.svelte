<script lang="ts">
	import { browser } from '$app/environment';

	export let width = '100%';
	export let height = '100%';
	export let values: string[];

	const ROTATION_OFFSET_PER_ENTRY = 20;

	let entries: { label: string; visible: boolean; rotation: number }[];
	$: entries = values.map((value, i) => ({
		label: value,
		visible: true,
		rotation: -1
	}));

	let pointerData = {
		pointerStart: -1,
		velocity: 0,
		rotOffset: 0
	};

	if (browser) {
		let count = 0;
		function animate() {
			count++;
			pointerData.rotOffset = Math.sin(count * 0.01) * 100;
			requestAnimationFrame(animate);
		}

		// requestAnimationFrame(animate);
	}

	function addRotationOffset(offset) {
		pointerData.rotOffset += offset;
		const rotationLimit = ROTATION_OFFSET_PER_ENTRY * (values.length - 1);
		if (pointerData.rotOffset < 0) pointerData.rotOffset = 0;
		if (pointerData.rotOffset > rotationLimit) pointerData.rotOffset = rotationLimit;
	}

	function onPointerDown(e) {
		pointerData.pointerStart = e.clientY;
	}
	function onPointerUp() {
		pointerData.pointerStart = -1;
	}
	function onPointerMove(e) {
		if (pointerData.pointerStart > -1) {
			const delta = e.clientY - pointerData.pointerStart;
			addRotationOffset(-delta);
			pointerData.pointerStart = e.clientY;
		}
	}

	// on pointer data change
	$: {
		entries.forEach((entry, i) => {
			entry.visible = true;
			entry.rotation = i * ROTATION_OFFSET_PER_ENTRY - pointerData.rotOffset;

			if (entry.rotation > 90 || entry.rotation < -90) entry.visible = false;
		});
		entries = entries;
	}
</script>

<svelte:window on:pointerup={onPointerUp} on:pointermove={onPointerMove} />
<div class="container" style:width style:height on:pointerdown={onPointerDown}>
	{#each entries as entry, i}
		{#if entry.visible}
			<p
				class="entry"
				style:transform={`translateY(-50%) rotateX(${-entry.rotation}deg) translate3d(0, 0, 80px)`}
			>
				{entry.label}
			</p>
		{/if}
	{/each}
</div>

<style>
	.container {
		background: #eee;
		position: relative;
	}

	.entry {
		position: absolute;
		will-change: transform;

		top: 50%;
		perspective: 200px;

		pointer-events: none;
	}
</style>
