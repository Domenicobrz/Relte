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
		rotOffset: 0,
		direction: 0
	};

	let animData: { state: 'idle' | 'snap' | 'inertia-rotation'; snapTo: number } = {
		state: 'idle',
		snapTo: -1
	};

	let entriesEls: HTMLParagraphElement[] = [];

	if (browser) {
		function animate() {
			// reset all animations if we're currently moving the wheel picker
			if (pointerData.pointerStart !== -1) {
				animData.state = 'idle';
				animData.snapTo = -1;
			}

			if (Math.abs(pointerData.velocity) > 0 && pointerData.pointerStart === -1) {
				animData.state = 'inertia-rotation';

				pointerData.velocity *= 0.95;
				addRotationOffset(-pointerData.velocity * deltaSpeedAdjust);

				if (Math.abs(pointerData.velocity) < 1) {
					pointerData.velocity = 0;
					animData.state = 'idle';
				}
			}

			const mod = pointerData.rotOffset % rotationOffsetPerEntry;
			const isUnstable = mod > 0;

			if (isUnstable && pointerData.pointerStart === -1 && animData.state == 'idle') {
				animData.state = 'snap';

				if (pointerData.direction < 0) {
					animData.snapTo = pointerData.rotOffset - mod + rotationOffsetPerEntry;
				} else {
					animData.snapTo = pointerData.rotOffset - mod;
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

	function getScale(entry: (typeof entries)[0]): number {
		return entry.scale + 0.1 * (1 - Math.min(Math.abs(entry.rotation), 5) / 5);
	}

	function getOpacity(offset: number): number {
		return 1 - (0.6 * Math.min(Math.abs(offset), 5)) / 5;
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
			pointerData.direction = Math.sign(delta);
			addRotationOffset(-delta * deltaSpeedAdjust);
			pointerData.pointerStart = y;
			pointerData.velocity = delta;
		}
	}

	// v v v these variables depend on the height of the container v v v
	let zTransl = 100;
	let perspective = 0;
	let rotationOffsetPerEntry = 0;
	let deltaSpeedAdjust = 1;
	let entryHeight = 0;
	// ^ ^ ^ these variables depend on the height of the container ^ ^ ^
	let containerEl: HTMLDivElement;
	let ro: ResizeObserver;

	onMount(() => {
		ro = new ResizeObserver((entries) => {
			for (let entry of entries) {
				perspective = entry.contentRect.height * 0.95;
				zTransl = entry.contentRect.height * 0.44;
				rotationOffsetPerEntry = 2200 / entry.contentRect.height;
				deltaSpeedAdjust = 60 / entry.contentRect.height;
			}
		});
		ro.observe(containerEl);
	});

	onDestroy(() => {
		if (ro) ro.disconnect();
	});

	// on pointer data change
	$: {
		entries.forEach((entry, i) => {
			entry.visible = true;
			entry.rotation = i * rotationOffsetPerEntry - pointerData.rotOffset;

			if (entry.rotation > 90 || entry.rotation < -90) entry.visible = false;
			entry.scale = 1;
		});
		entries = entries;
	}

	$: {
		// we need to calculate the height of the selection-highlight whenever entriesEls
		// changes
		let biggestHeight = 0;
		entriesEls.forEach((el) => {
			if (!el) return;

			const currHeight = el.getBoundingClientRect().height;
			if (currHeight > biggestHeight) biggestHeight = currHeight;
		});
		entryHeight = biggestHeight;
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
	style:perspective={perspective + 'px'}
	on:pointerdown={onPointerDown}
	on:touchstart={onPointerDown}
>
	<div style:height={entryHeight + 5 + 'px'} class="selection-highlight" />
	{#each entries as entry, i}
		{#if entry.visible}
			<p
				class="entry"
				bind:this={entriesEls[i]}
				style:opacity={getOpacity(entry.rotation)}
				style:transform={`translateY(-50%) rotateX(${-entry.rotation}deg) translate3d(0, 0, ${zTransl}px) scale(${getScale(
					entry
				)})`}
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
		will-change: opacity;

		margin: 0;

		user-select: none;
		pointer-events: none;

		text-align: center;

		backface-visibility: hidden;

		font-size: 10px;
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

	.selection-highlight {
		width: 100%;
		position: absolute;
		top: 50%;
		transform: translateY(-50%);
		background: #ddd;
	}
</style>
