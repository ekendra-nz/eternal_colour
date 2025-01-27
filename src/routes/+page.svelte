<script lang="ts">
	import { onMount } from 'svelte';

	// Define the canvas element and its size
	let canvas: HTMLCanvasElement | null = null;

	let size = 100; // Diameter of the color wheel
	let percentOfWindow = 0.7;
	onMount(() => {
		const windowHeight = window.innerHeight;
		const windowWidth = window.innerWidth;
		const height = Math.round(windowHeight * percentOfWindow);
		const width = Math.round(windowWidth * percentOfWindow);
		// size = Math.round(Math.min(height, width) / 100) * 100;
		size = Math.min(height, width);
		console.log('size', size, 'height', height, 'width', width);

		if (canvas) {
			canvas.width = size;
			canvas.height = size;

			const ctx = canvas.getContext('2d');
			if (!ctx) return;

			const radius = size / 2;
			const imageData = ctx.createImageData(size, size);

			// Loop through each pixel in the canvas
			for (let y = -radius; y < radius; y++) {
				for (let x = -radius; x < radius; x++) {
					const distance = Math.sqrt(x * x + y * y);

					// Only color pixels within the circle
					if (distance <= radius) {
						const angle = Math.atan2(y, x) + Math.PI; // Range: [0, 2π]
						const hue = (angle / (2 * Math.PI)) * 360; // Convert angle to hue
						const saturation = distance / radius; // Saturation based on distance
						const value = 1; // Full value

						// Convert HSV to RGB
						const [r, g, b] = hsvToRgb(hue, saturation, value);

						const pixelIndex = ((y + radius) * size + (x + radius)) * 4;
						imageData.data[pixelIndex] = r;
						imageData.data[pixelIndex + 1] = g;
						imageData.data[pixelIndex + 2] = b;
						imageData.data[pixelIndex + 3] = 255; // Alpha channel
					}
				}
			}

			// Render the generated image onto the canvas
			ctx.putImageData(imageData, 0, 0);
		}
	});

	/**
	 * Convert HSV to RGB
	 * @param h - Hue (0-360 degrees)
	 * @param s - Saturation (0-1)
	 * @param v - Value (0-1)
	 * @returns [red, green, blue] in 0-255 range
	 */
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

<div class="m-6 flex justify-center">
	<canvas bind:this={canvas}></canvas>
</div>

<style>
	canvas {
		border: 10px solid black;
		border-radius: 50%;
	}
</style>
