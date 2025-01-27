<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	let canvas: HTMLCanvasElement | null = null;
	const luminosity = writable(0); // Luminosity (-1 to 1)
	let size = 100; // Canvas size
	let percentOfWindow = 0.7; // Scale relative to window
	let label = ''; // Display label

	onMount(() => {
		const windowHeight = window.innerHeight;
		const windowWidth = window.innerWidth;
		const height = Math.round(windowHeight * percentOfWindow);
		const width = Math.round(windowWidth * percentOfWindow);
		size = Math.min(height, width);

		// Update canvas whenever luminosity changes
		const unsubscribe = luminosity.subscribe((lum) => {
			if (canvas) {
				canvas.width = size;
				canvas.height = size;

				const ctx = canvas.getContext('2d');
				if (!ctx) return;

				const radius = size / 2;
				const extendedRadius = radius + 1; // Slightly extend radius to eliminate white edge
				const imageData = ctx.createImageData(size, size);

				// Update label with rounded values
				label =
					lum < 0
						? `${Math.round(Math.abs(lum) * 100)}% Black Influence`
						: lum > 0
							? `${Math.round(lum * 100)}% White Influence`
							: `0% Black or White Influence (Pure Colors)`;

				// Loop through pixels
				for (let y = -radius; y < radius; y++) {
					for (let x = -radius; x < radius; x++) {
						const distance = Math.sqrt(x * x + y * y);

						// Only render within the extended circle
						if (distance <= extendedRadius) {
							const angle = Math.atan2(y, x) + Math.PI; // Range: [0, 2π]
							let hue = (angle / (2 * Math.PI)) * 360;
							if (hue >= 360) hue = 0;

							const saturation = Math.min(1, distance / radius);
							const value = 1; // Full value

							let [r, g, b] = hsvToRgb(hue, saturation, value);

							// Adjust for luminosity
							if (lum > 0) {
								r = Math.round(r * (1 - lum) + 255 * lum);
								g = Math.round(g * (1 - lum) + 255 * lum);
								b = Math.round(b * (1 - lum) + 255 * lum);
							} else if (lum < 0) {
								r = Math.round(r * (1 + lum));
								g = Math.round(g * (1 + lum));
								b = Math.round(b * (1 + lum));
							}

							const pixelIndex = ((y + radius) * size + (x + radius)) * 4;
							imageData.data[pixelIndex] = r;
							imageData.data[pixelIndex + 1] = g;
							imageData.data[pixelIndex + 2] = b;
							imageData.data[pixelIndex + 3] = 255;
						}
					}
				}

				// Render the generated image onto the canvas
				ctx.putImageData(imageData, 0, 0);
			}
		});

		return unsubscribe;
	});

	function hsvToRgb(h: number, s: number, v: number): [number, number, number] {
		const c = v * s;
		const x = c * (1 - Math.abs(((h / 60) % 2) - 1));
		const m = v - c;

		let r = 0,
			g = 0,
			b = 0;

		if (0 <= h && h < 60) {
			r = c;
			g = x;
			b = 0;
		} else if (60 <= h && h < 120) {
			r = x;
			g = c;
			b = 0;
		} else if (120 <= h && h < 180) {
			r = 0;
			g = c;
			b = x;
		} else if (180 <= h && h < 240) {
			r = 0;
			g = x;
			b = c;
		} else if (240 <= h && h < 300) {
			r = x;
			g = 0;
			b = c;
		} else if (300 <= h && h < 360) {
			r = c;
			g = 0;
			b = x;
		}

		return [Math.round((r + m) * 255), Math.round((g + m) * 255), Math.round((b + m) * 255)];
	}
</script>

<div class="m-6 flex flex-col items-center">
	<p class="text-lg font-medium mb-2">{label}</p>
	<input
		id="luminositySlider"
		type="range"
		min="-1"
		max="1"
		step="0.01"
		bind:value={$luminosity}
		class="w-full mb-6"
	/>
	<canvas bind:this={canvas}></canvas>
</div>

<style>
	canvas {
		border: 10px solid black;
		border-radius: 50%;
	}
</style>
