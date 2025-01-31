<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	let canvas: HTMLCanvasElement | null = null;
	const luminosity = writable(0); // Slider value (0 to 1)
	let size = 100; // Canvas size
	let percentOfWindow = 0.7; // Scale relative to window
	let label = ''; // Display label

	// Convert Hex to HSV
	function hexToHsv(hex: string): { h: number; s: number; v: number } {
		let r: number = 0,
			g: number = 0,
			b: number = 0;

		// 3 digits
		if (hex.length === 4) {
			r = parseInt(hex[1] + hex[1], 16);
			g = parseInt(hex[2] + hex[2], 16);
			b = parseInt(hex[3] + hex[3], 16);
		}
		// 6 digits
		if (hex.length === 7) {
			r = parseInt(hex[1] + hex[2], 16);
			g = parseInt(hex[3] + hex[4], 16);
			b = parseInt(hex[5] + hex[6], 16);
		}

		// Normalize RGB values to the range 0-1
		r /= 255;
		g /= 255;
		b /= 255;

		// Find the maximum and minimum RGB values
		const max = Math.max(r, g, b);
		const min = Math.min(r, g, b);
		const delta = max - min;

		let h = 0;
		let s = max === 0 ? 0 : delta / max;
		let v = max;

		if (delta !== 0) {
			if (max === r) {
				h = (g - b) / delta;
			} else if (max === g) {
				h = 2 + (b - r) / delta;
			} else {
				h = 4 + (r - g) / delta;
			}
			h *= 60;
			if (h < 0) h += 360;
		}

		return { h, s, v };
	}

	// Monk skin tones (with #f7ede4 for Type A)
	const skinTonesHSV = [
		{ h: [40, 50], s: [0, 0.1], v: [0, 10.97], label: 'Type A', range: 0.1 },
		{ h: [10, 20], s: [0.2, 0.3], v: [0.9, 1.0], label: 'Type B', range: 0.1 },
		{ h: [15, 25], s: [0.3, 0.4], v: [0.85, 0.95], label: 'Type C', range: 0.1 },
		{ h: [20, 30], s: [0.4, 0.5], v: [0.75, 0.85], label: 'Type D', range: 0.1 },
		{ h: [25, 35], s: [0.5, 0.6], v: [0.65, 0.75], label: 'Type E', range: 0.1 },
		{ h: [30, 40], s: [0.6, 0.7], v: [0.55, 0.65], label: 'Type F', range: 0.1 },
		{ h: [35, 45], s: [0.7, 0.8], v: [0.45, 0.55], label: 'Type G', range: 0.1 },
		{ h: [40, 50], s: [0.8, 0.9], v: [0.35, 0.45], label: 'Type H', range: 0.1 },
		{ h: [45, 55], s: [0.9, 1.0], v: [0.25, 0.35], label: 'Type I', range: 0.1 },
		// Explicitly set Type J to have the desired RGB
		{
			h: [50, 60],
			s: [0.9, 1.0],
			v: [0.15, 0.25],
			label: 'Type J',
			range: 0.1,
			customRgb: [42, 36, 32]
		}
	];

	// Interpolate between two values
	function interpolateValue(start: number, end: number, factor: number): number {
		return start + factor * (end - start);
	}

	// Interpolate between two HSV ranges
	function interpolateHSV(
		start: { h: number[]; s: number[]; v: number[] },
		end: { h: number[]; s: number[]; v: number[] },
		factor: number
	): { h: number; s: number; v: number } {
		return {
			h: interpolateValue(start.h[0], end.h[1], factor),
			s: interpolateValue(start.s[0], end.s[1], factor),
			v: interpolateValue(start.v[0], end.v[1], factor)
		};
	}

	// Convert HSV to RGB
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
				const extendedRadius = radius + 1;
				const imageData = ctx.createImageData(size, size);

				// Determine skin tone based on slider value
				let adjustedLum = Math.max(0, lum - 0.05) / 0.95; // Skip first 5%
				let cumulativeRange = 0;
				let toneIndex = 0;

				for (let i = 0; i < skinTonesHSV.length; i++) {
					cumulativeRange += skinTonesHSV[i].range;
					if (lum <= cumulativeRange) {
						toneIndex = i;
						break;
					}
				}

				const rangeStart = cumulativeRange - skinTonesHSV[toneIndex].range;
				const factor = (lum - rangeStart) / skinTonesHSV[toneIndex].range;

				// Fix: when luminosity is all the way to the right (1.0)
				if (lum === 1) {
					toneIndex = skinTonesHSV.length - 1;
				}

				// Explicit check for Type J
				let skinToneHSV;
				let r = 0,
					g = 0,
					b = 0; // Declare r, g, b for Type J RGB handling
				if (skinTonesHSV[toneIndex].customRgb) {
					// Use custom RGB for Type J
					skinToneHSV = { h: 0, s: 0, v: 0 };
					[r, g, b] = skinTonesHSV[toneIndex].customRgb; // Assign custom RGB
				} else {
					skinToneHSV = interpolateHSV(
						skinTonesHSV[toneIndex],
						skinTonesHSV[Math.min(toneIndex + 1, skinTonesHSV.length - 1)],
						factor
					);
					[r, g, b] = hsvToRgb(skinToneHSV.h, skinToneHSV.s, skinToneHSV.v); // Use interpolation result
				}

				// Update label
				if (lum <= 0.05) {
					label = `${skinTonesHSV[0].label}`;
				} else {
					label = `${skinTonesHSV[toneIndex].label} - RGB(${r}, ${g}, ${b})`;
				}

				// Loop through pixels
				for (let y = -radius; y < radius; y++) {
					for (let x = -radius; x < radius; x++) {
						const distance = Math.sqrt(x * x + y * y);

						if (distance <= extendedRadius) {
							const angle = Math.atan2(y, x) + Math.PI;
							let hue = (angle / (2 * Math.PI)) * 360;
							if (hue >= 360) hue = 0;

							const saturation = Math.min(1, distance / radius);
							const value = 1;

							let [pixelR, pixelG, pixelB] = hsvToRgb(hue, saturation, value);

							pixelR = Math.round(pixelR * (1 - lum) + r * lum);
							pixelG = Math.round(pixelG * (1 - lum) + g * lum);
							pixelB = Math.round(pixelB * (1 - lum) + b * lum);

							const pixelIndex = ((y + radius) * size + (x + radius)) * 4;
							imageData.data[pixelIndex] = pixelR;
							imageData.data[pixelIndex + 1] = pixelG;
							imageData.data[pixelIndex + 2] = pixelB;
							imageData.data[pixelIndex + 3] = 255;
						}
					}
				}

				ctx.putImageData(imageData, 0, 0);
			}
		});

		return unsubscribe;
	});
</script>

<div class="container">
	<div class="label">{label}</div>

	<div class="slider-container">
		<div class="gradient-bar">
			<div class="gradient"></div>
		</div>
		<div class="labels">
			<span>Type A</span>
			<span>Type B</span>
			<span>Type C</span>
			<span>Type D</span>
			<span>Type E</span>
			<span>Type F</span>
			<span>Type G</span>
			<span>Type H</span>
			<span>Type I</span>
			<span>Type J</span>
		</div>
		<input class="slider" type="range" min="0" max="1" step="0.01" bind:value={$luminosity} />
	</div>

	<canvas bind:this={canvas}></canvas>
</div>

<style>
	.container {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 20px;
		margin: 20px;
	}

	.label {
		font-size: 1.2em;
		font-weight: bold;
		text-align: center;
	}

	.slider-container {
		width: 80%;
		position: relative;
	}

	.gradient-bar {
		width: 100%;
		height: 20px;
		position: relative;
		display: flex;
		align-items: center;
	}

	.gradient {
		position: absolute;
		width: 100%;
		height: 100%;
		background: linear-gradient(
			to right,
			#f7ede4 0%,
			#f3e7da 10%,
			#f6ead0 20%,
			#ead9bb 30%,
			#d7bd96 40%,
			#9f7d54 50%,
			#815d44 60%,
			#604234 70%,
			#3a312a 80%,
			#2a2420 90%
		);
	}

	.labels {
		display: flex;
		justify-content: space-between;
		margin-top: 5px;
		font-size: 0.9em;
		font-weight: bold;
		color: #f6e2b1;
	}

	.slider {
		width: 100%;
		margin-top: 10px;
	}
</style>
