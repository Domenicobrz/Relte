<script context="module" lang="ts">
	export type SelectedEventDetail = { value: string; index: number };
</script>

<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	const dispatch = createEventDispatcher<{ selected: SelectedEventDetail }>();

	export let width = '100%';
	export let height = '100%';
	export let values: string[];
	export let theme: 'default' | 'inset' | 'none' | 'bard' = 'default';
	export let selected: string = values[0];

	let selectedIndex = -1;
	$: {
		// svelte works in such a way that if selectedIndex remains the same,
		// this snippet won't run again, as in, if the user re-selects the same element
		// this snippet is not going to run again

		// for the initial selection on component mount,
		// don't animate to the newly selected index
		if (selectedIndex === -1) {
			selectedIndex = values.indexOf(selected);
			pointerData.rotOffset = selectedIndex * rotationOffsetPerEntry;
		} else {
			// for new selections, animate to the new selected index
			selectedIndex = values.indexOf(selected);
			selectAndSnap(selectedIndex);
		}
	}
	$: themeName = theme + '-theme';

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
			// just doing mod > 0 isn't enough due to floating point precision
			// issues where the modulo could stay fixed at e.g. 0.9999999999986
			const isUnstable = mod > 0.0001 && mod < rotationOffsetPerEntry * 0.9999;

			if (isUnstable && pointerData.pointerStart === -1 && animData.state == 'idle') {
				animData.state = 'snap';

				const t = mod / rotationOffsetPerEntry;

				let currentIndex = Math.round((pointerData.rotOffset - mod) / rotationOffsetPerEntry);
				currentIndex = Math.min(currentIndex, values.length - 1);

				if (pointerData.direction < 0) {
					if (t > 0.3) {
						selectAndSnap(currentIndex + 1);
					} else {
						selectAndSnap(currentIndex);
					}
				} else {
					if (t < 0.7) {
						selectAndSnap(currentIndex);
					} else {
						selectAndSnap(currentIndex + 1);
					}
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
		pointerData.direction = 0;
	}
	function onPointerUp() {
		pointerData.pointerStart = -1;
	}
	function onPointerMove(e: MouseEvent | TouchEvent) {
		if (pointerData.pointerStart > -1) {
			let y = 'touches' in e ? e.touches[0].clientY : e.clientY;

			const delta = y - pointerData.pointerStart;
			// here we're summing the direction instead of assigning -1 or +1
			// because otherwise we can often get "0" as a result of small movements
			// and that can break the modulo calculations to determine the snapping
			// point
			pointerData.direction += Math.sign(delta);
			addRotationOffset(-delta * deltaSpeedAdjust);
			pointerData.pointerStart = y;
			pointerData.velocity = delta;
		}
	}

	function selectAndSnap(newIndex: number) {
		animData.state = 'snap';
		animData.snapTo = newIndex * rotationOffsetPerEntry;

		dispatch('selected', { value: values[newIndex], index: newIndex });
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
		entryHeight = biggestHeight + 2;
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
	class="container {themeName}"
	style:width
	style:height
	style:perspective={perspective + 'px'}
	on:pointerdown={onPointerDown}
	on:touchstart={onPointerDown}
>
	<div style:height={entryHeight + 5 + 'px'} class="selection-highlight {themeName}" />
	{#each entries as entry, i}
		{#if entry.visible}
			<p
				class="entry {themeName}"
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
	<div class="top-curtain curtain {themeName}" />
	<div class="bottom-curtain curtain {themeName}" />
</div>

<style>
	@import './no-theme.scss';
	@import './default-theme.scss';
	@import './inset-theme.scss';
	@import './bard-theme.scss';
</style>
