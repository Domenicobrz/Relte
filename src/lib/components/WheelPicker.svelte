<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	export let width = '100%';
	export let height = '100%';
	export let values: string[];

	let entries: { label: string; visible: boolean; rotation: number; scale: number }[];
	$: entries = values.map((value, i) => ({
		label: value,
		visible: true,
		rotation: -1,
		scale: 1
	}));

	let pointerData = {
		pointerStart: -1,
		velocity: 0,
		rotOffset: 0
	};

	let animData: { state: 'idle' | 'snap'; snapTo: number } = {
		state: 'idle',
		snapTo: -1
	};

	if (browser) {
		function animate() {
			// reset all animations if we're currently moving the wheel picker
			if (pointerData.pointerStart !== -1) {
				animData.state = 'idle';
				animData.snapTo = -1;
			}

			const mod = pointerData.rotOffset % rotationOffsetPerEntry;
			const isUnstable = mod > 0;

			if (isUnstable && pointerData.pointerStart === -1 && animData.state == 'idle') {
				animData.state = 'snap';

				const at = mod;
				// [0 ... 1]
				const t = at / rotationOffsetPerEntry;
				if (t < 0.5) {
					animData.snapTo = pointerData.rotOffset - mod;
				} else {
					animData.snapTo = pointerData.rotOffset - mod + rotationOffsetPerEntry;
				}
			}

			if (animData.state == 'snap') {
				let delta = animData.snapTo - pointerData.rotOffset;
				pointerData.rotOffset += delta * 0.075;

				if (Math.abs(delta) < 0.1) {
					pointerData.rotOffset = animData.snapTo;
					animData.state = 'idle';
					animData.snapTo = -1;
				}
			}

			requestAnimationFrame(animate);
		}

		requestAnimationFrame(animate);
	}

	function addRotationOffset(offset: number) {
		pointerData.rotOffset += offset;
		const rotationLimit = rotationOffsetPerEntry * (values.length - 1);
		if (pointerData.rotOffset < 0) pointerData.rotOffset = 0;
		if (pointerData.rotOffset > rotationLimit) pointerData.rotOffset = rotationLimit;
	}

	function onPointerDown(e: PointerEvent | TouchEvent) {
		let y = 'touches' in e ? e.touches[0].clientY : e.clientY;
		pointerData.pointerStart = y;
	}
	function onPointerUp() {
		pointerData.pointerStart = -1;
	}
	function onPointerMove(e: MouseEvent | TouchEvent) {
		if (pointerData.pointerStart > -1) {
			let y = 'touches' in e ? e.touches[0].clientY : e.clientY;

			const delta = y - pointerData.pointerStart;
			addRotationOffset(-delta * deltaSpeedAdjust);
			pointerData.pointerStart = y;
		}
	}

	// v v v these variables depend on the height of the container v v v
	let perspectiveHeight = 100;
	let rotationOffsetPerEntry = 0;
	let deltaSpeedAdjust = 1;
	// ^ ^ ^ these variables depend on the height of the container ^ ^ ^
	let containerEl: HTMLDivElement;
	let ro: ResizeObserver; // Reference for ResizeObserver

	onMount(() => {
		ro = new ResizeObserver((entries) => {
			for (let entry of entries) {
				perspectiveHeight = entry.contentRect.height * 0.5;
				rotationOffsetPerEntry = 2500 / entry.contentRect.height;
				deltaSpeedAdjust = 100 / entry.contentRect.height;
			}
		});

		ro.observe(containerEl);
	});

	onDestroy(() => {
		// Disconnect ResizeObserver when the component is destroyed
		if (ro) {
			ro.disconnect();
		}
	});

	// on pointer data change
	$: {
		entries.forEach((entry, i) => {
			entry.visible = true;
			entry.rotation = i * rotationOffsetPerEntry - pointerData.rotOffset;

			if (entry.rotation > 90 || entry.rotation < -90) entry.visible = false;
			entry.scale = 1 - Math.pow(Math.abs(entry.rotation) / 90, 2) * 0.4;
		});
		entries = entries;
	}
</script>

<svelte:window
	on:pointerup={onPointerUp}
	on:pointermove={onPointerMove}
	on:touchend={onPointerUp}
	on:touchmove={onPointerMove}
/>
<div
	bind:this={containerEl}
	class="container"
	style:width
	style:height
	on:pointerdown={onPointerDown}
	on:touchstart={onPointerDown}
>
	{#each entries as entry, i}
		{#if entry.visible}
			<p
				class="entry"
				style:transform={`translateY(-50%) rotateX(${-entry.rotation}deg) translate3d(0, 0, ${perspectiveHeight}px) scale(${
					entry.scale
				}) `}
			>
				{entry.label}
			</p>
		{/if}
	{/each}
	<div class="top-curtain curtain" />
	<div class="bottom-curtain curtain" />
</div>

<style>
	.container {
		background: #eee;
		position: relative;
		user-select: none;
	}

	.entry {
		width: 100%;
		top: 50%;
		position: absolute;
		will-change: transform, opacity;

		margin: 0;

		user-select: none;
		pointer-events: none;

		text-align: center;
	}

	.curtain {
		position: absolute;
		width: 100%;
		height: 45px;
	}

	.top-curtain {
		top: 0;
		background: linear-gradient(to bottom, #eee 0%, transparent);
	}

	.bottom-curtain {
		bottom: 0;
		background: linear-gradient(to top, #eee 0%, transparent);
	}
</style>
