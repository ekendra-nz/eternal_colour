<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	let canvas: HTMLCanvasElement | null = null;
	const luminosity = writable(0); // Luminosity (0 to 1)
	let size = 100; // Canvas size
	let percentOfWindow = 0.7; // Scale relative to window
	let label = ''; // Display label

	// Fitzpatrick skin tones (RGB values)
	const skinTones = [
		{ r: 255, g: 224, b: 189, label: 'No Overlay (Pure Wheel)' }, // No overlay (pure color wheel)
		{ r: 223, g: 170, b: 129, label: 'F I (Light Skin)' }, // Fitzpatrick I (Light Skin)
		{ r: 173, g: 115, b: 91, label: 'F II (Medium Light Skin)' }, // Fitzpatrick II (Medium Light Skin)
		{ r: 139, g: 87, b: 63, label: 'F III (Medium Skin)' }, // Fitzpatrick III (Medium Skin)
		{ r: 105, g: 63, b: 47, label: 'F IV (Tan Skin)' }, // Fitzpatrick IV (Tan Skin)
		{ r: 80, g: 47, b: 34, label: 'F V (Dark Skin)' } // Fitzpatrick V (Dark Skin)
	];

	// Function to interpolate between two colors
	function interpolateColor(
		start: { r: number; g: number; b: number },
		end: { r: number; g: number; b: number },
		factor: number
	): { r: number; g: number; b: number } {
		return {
			r: Math.round(start.r + factor * (end.r - start.r)),
			g: Math.round(start.g + factor * (end.g - start.g)),
			b: Math.round(start.b + factor * (end.b - start.b))
		};
	}

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

				// Update label with skin tone description
				const sectionIndex = Math.min(
					Math.floor(lum * (skinTones.length - 1)),
					skinTones.length - 1
				);
				label = `${skinTones[sectionIndex].label}`;

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

							// Interpolate skin tone based on luminosity
							const toneIndex = Math.min(
								Math.floor(lum * (skinTones.length - 1)),
								skinTones.length - 1
							);
							const nextToneIndex = Math.min(toneIndex + 1, skinTones.length - 1);
							const factor = lum * (skinTones.length - 1) - toneIndex;

							const skinTone = interpolateColor(
								skinTones[toneIndex],
								skinTones[nextToneIndex],
								factor
							);

							r = Math.round(r * (1 - lum) + skinTone.r * lum);
							g = Math.round(g * (1 - lum) + skinTone.g * lum);
							b = Math.round(b * (1 - lum) + skinTone.b * lum);

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

	<!-- Skin Tone Color Bar with Labels -->
	<div class="w-full mt-4 h-8 flex relative">
		<div style="width: 5%; background-color: #fff;">
			<p
				class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-black text-xs"
			>
				No Overlay
			</p>
		</div>
		{#each skinTones.slice(1) as tone, i (tone.label)}
			<div class="flex-1 relative" style="background-color: rgb({tone.r}, {tone.g}, {tone.b})">
				<p
					class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 text-white text-xs font-bold"
				>
					{tone.label}
				</p>
			</div>
		{/each}
	</div>

	<!-- Slider -->
	<input
		id="luminositySlider"
		type="range"
		min="0"
		max="1"
		step="0.01"
		bind:value={$luminosity}
		class="w-full mb-6"
	/>

	<!-- Canvas for color wheel -->
	<canvas bind:this={canvas}></canvas>
</div>

<style>
	canvas {
		border: 10px solid black;
		border-radius: 50%;
	}
	.slider-container {
		width: 100%;
		display: flex;
		justify-content: space-between;
		padding: 0 20px;
	}
	.slider-label {
		display: flex;
		justify-content: space-between;
		width: 100%;
	}
</style>
