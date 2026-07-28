<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
	import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
	import { STLExporter } from 'three/examples/jsm/exporters/STLExporter.js';

	let canvas: HTMLCanvasElement;
	let topText = 'TOP TEXT';
	let bottomText = 'BOTTOM TEXT';
	let circleColor = '#FF0000'; // Red by default
	let topFontError: string | null = null;
	let bottomFontError: string | null = null;

	// Status presets
	const presets = [
		{ name: 'Do not disturb', text: 'Do not disturb', color: '#FF0000' },
		{ name: 'Ask', text: 'Ask', color: '#FFA500' },
		{ name: 'Active', text: 'Active', color: '#00AA00' },
		{ name: 'Join Me', text: 'Join Me', color: '#00CCFF' }
	];

	let scene: THREE.Scene;
	let camera: THREE.PerspectiveCamera;
	let renderer: THREE.WebGLRenderer;
	let controls: OrbitControls;
	let badgeGroup: THREE.Group;
	let fontLoader = new FontLoader();
	let topFont: any = null;
	let bottomFont: any = null;
	let topTextMesh: THREE.Mesh | null = null;
	let bottomTextMesh: THREE.Mesh | null = null;
	let bottomCircleMesh: THREE.Mesh | null = null;
	let circleMaterial: THREE.MeshStandardMaterial;

	const topPlateW = 105 - 0.5; // 104.5
	const topPlateH = 16 - 0.5; // 15.5
	const bottomPlateW = 105 - 0.5; // 104.5
	const bottomPlateH = 10 - 0.5; // 9.5
	const plateD = 2;
	const textD = 2; // Extrusion depth
	const gap = 3;

	const topY = topPlateH / 2 + gap / 2;
	const bottomY = -(bottomPlateH / 2 + gap / 2);

	const plateMaterial = new THREE.MeshStandardMaterial({ color: 0x444444 });
	const textMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff });

	onMount(() => {
		initScene();
		// Load default font for both top and bottom
		fontLoader.load('/fonts/helvetiker_regular.typeface.json', (font) => {
			topFont = font;
			bottomFont = font;
			updateTopText();
			updateBottomText();
		});

		return () => {
			renderer.dispose();
		};
	});

	function initScene() {
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0xefefef);

		const aspect = window.innerWidth / window.innerHeight;
		camera = new THREE.PerspectiveCamera(50, aspect, 0.1, 1000);
		camera.position.set(0, 0, 150);

		renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);

		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;

		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);

		const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
		dirLight.position.set(50, 50, 100);
		scene.add(dirLight);

		// Initialize circle material with the current color
		circleMaterial = new THREE.MeshStandardMaterial({ color: circleColor });

		badgeGroup = new THREE.Group();
		scene.add(badgeGroup);

		// Create Plates
		const topPlateGeo = new THREE.BoxGeometry(topPlateW, topPlateH, plateD);
		const topPlateMesh = new THREE.Mesh(topPlateGeo, plateMaterial);
		topPlateMesh.position.set(0, topY, 0);
		badgeGroup.add(topPlateMesh);

		const bottomPlateGeo = new THREE.BoxGeometry(bottomPlateW, bottomPlateH, plateD);
		const bottomPlateMesh = new THREE.Mesh(bottomPlateGeo, plateMaterial);
		bottomPlateMesh.position.set(0, bottomY, 0);
		badgeGroup.add(bottomPlateMesh);

		animate();
		window.addEventListener('resize', onWindowResize);
	}

	function updateTopText() {
		if (!topFont) return;
		if (topTextMesh) {
			badgeGroup.remove(topTextMesh);
			topTextMesh.geometry.dispose();
		}

		const size = topPlateH * 0.6;
		const textGeo = new TextGeometry(topText, {
			font: topFont,
			size: size,
			depth: textD,
			curveSegments: 12,
			bevelEnabled: false
		});
		textGeo.computeBoundingBox();
		const tBbox = textGeo.boundingBox!;
		const tCenterX = (tBbox.max.x + tBbox.min.x) / 2;

		// Calculate fixed vertical center using a reference string so baseline doesn't jump
		const refGeo = new TextGeometry('My', {
			font: topFont,
			size: size,
			depth: textD,
			curveSegments: 1,
			bevelEnabled: false
		});
		refGeo.computeBoundingBox();
		const refCenterY = (refGeo.boundingBox!.max.y + refGeo.boundingBox!.min.y) / 2;
		refGeo.dispose();

		topTextMesh = new THREE.Mesh(textGeo, textMaterial);
		// Center text on plate
		topTextMesh.position.set(-tCenterX, topY - refCenterY, plateD / 2);
		badgeGroup.add(topTextMesh);
	}

	function updateBottomText() {
		if (!bottomFont) return;
		if (bottomTextMesh) {
			badgeGroup.remove(bottomTextMesh);
			bottomTextMesh.geometry.dispose();
		}
		if (bottomCircleMesh) {
			badgeGroup.remove(bottomCircleMesh);
			bottomCircleMesh.geometry.dispose();
		}
		if (!bottomText) return;

		const size = bottomPlateH * 0.6;
		const textGeo = new TextGeometry(bottomText, {
			font: bottomFont,
			size: size,
			depth: textD,
			curveSegments: 12,
			bevelEnabled: false
		});
		textGeo.computeBoundingBox();
		const tBbox = textGeo.boundingBox!;
		const tWidth = tBbox.max.x - tBbox.min.x;

		// Calculate fixed vertical center using a reference string so baseline doesn't jump
		const refGeo = new TextGeometry('My', {
			font: bottomFont,
			size: size,
			depth: textD,
			curveSegments: 1,
			bevelEnabled: false
		});
		refGeo.computeBoundingBox();
		const refCenterY = (refGeo.boundingBox!.max.y + refGeo.boundingBox!.min.y) / 2;
		refGeo.dispose();

		const circleRadius = 3.5;
		const circleDiameter = circleRadius * 2;
		const gapBetween = 10; // 10mm gap
		const totalWidth = circleDiameter + gapBetween + tWidth;

		const startX = -totalWidth / 2;

		// Circle
		const circleGeo = new THREE.CylinderGeometry(circleRadius, circleRadius, textD, 32);
		bottomCircleMesh = new THREE.Mesh(circleGeo, circleMaterial);
		bottomCircleMesh.rotation.x = Math.PI / 2;
		bottomCircleMesh.position.set(startX + circleRadius, bottomY, plateD / 2 + textD / 2);
		badgeGroup.add(bottomCircleMesh);

		// Text
		bottomTextMesh = new THREE.Mesh(textGeo, textMaterial);
		bottomTextMesh.position.set(
			startX + circleDiameter + gapBetween - tBbox.min.x,
			bottomY - refCenterY,
			plateD / 2
		);
		badgeGroup.add(bottomTextMesh);
	}

	function updateCircleColor() {
		if (circleMaterial) {
			circleMaterial.color.setStyle(circleColor);
		}
	}

	function applyPreset(preset: (typeof presets)[0]) {
		circleColor = preset.color;
		bottomText = preset.text;
		updateCircleColor();
		updateBottomText();
	}

	function onWindowResize() {
		if (!camera || !renderer) return;
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}

	function animate() {
		requestAnimationFrame(animate);
		controls.update();
		renderer.render(scene, camera);
	}

	function exportSTL() {
		const exporter = new STLExporter();
		const stlString = exporter.parse(badgeGroup);
		const blob = new Blob([stlString], { type: 'text/plain' });
		const url = URL.createObjectURL(blob);
		const link = document.createElement('a');
		link.style.display = 'none';
		link.href = url;
		link.download = 'badge.stl';
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
	}

	function handleTopFontChange(e: Event) {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			topFontError = null;
			const reader = new FileReader();
			reader.onload = (event) => {
				try {
					const json = JSON.parse(event.target?.result as string);
					topFont = fontLoader.parse(json);
					updateTopText();
				} catch (e) {
					topFontError = 'Invalid font file';
					console.error('Font parsing error:', e);
				}
			};
			reader.readAsText(file);
		}
	}

	function handleBottomFontChange(e: Event) {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			bottomFontError = null;
			const reader = new FileReader();
			reader.onload = (event) => {
				try {
					const json = JSON.parse(event.target?.result as string);
					bottomFont = fontLoader.parse(json);
					updateBottomText();
				} catch (e) {
					bottomFontError = 'Invalid font file';
					console.error('Font parsing error:', e);
				}
			};
			reader.readAsText(file);
		}
	}
</script>

<div class="relative h-screen w-screen overflow-hidden bg-base-200">
	<div class="absolute inset-4 right-auto z-10 flex max-h-[calc(100vh-2rem)]">
		<div class="card w-72 overflow-y-auto bg-base-100/90 shadow-xl backdrop-blur-sm sm:w-96 md:w-[28rem]">
			<div class="card-body gap-3 p-4 sm:gap-4 sm:p-6">
				<h2 class="card-title text-base-content text-lg sm:text-xl">EPIC Badge editor</h2>

				<div class="form-control w-full">
					<label class="label py-2 sm:py-3">
						<span class="label-text text-sm font-bold sm:text-base">Top Text:</span>
					</label>
					<input
						class="input-bordered input input-sm w-full sm:input-md"
						type="text"
						bind:value={topText}
						on:input={updateTopText}
					/>
				</div>

				<div class="form-control w-full">
					<label class="label py-2 sm:py-3">
						<span class="label-text text-sm font-bold sm:text-base">Top Font (JSON):</span>
					</label>
					<input
						class="file-input-bordered file-input file-input-sm w-full sm:file-input-md"
						type="file"
						accept=".json"
						on:change={handleTopFontChange}
					/>
				</div>

				{#if topFontError}
					<div class="alert alert-warning">
						<span>{topFontError}</span>
					</div>
				{/if}
				<div class="divider my-1 sm:my-2"></div>

				<div class="form-control w-full">
					<label class="label py-2 sm:py-3">
						<span class="label-text text-sm font-bold sm:text-base">Bottom Text:</span>
					</label>
					<input
						class="input-bordered input input-sm w-full sm:input-md"
						type="text"
						bind:value={bottomText}
						on:input={updateBottomText}
					/>
				</div>

				<div class="form-control w-full">
					<label class="label py-2 sm:py-3">
						<span class="label-text text-sm font-bold sm:text-base">Bottom Font (JSON):</span>
					</label>
					<input
						class="file-input-bordered file-input file-input-sm w-full sm:file-input-md"
						type="file"
						accept=".json"
						on:change={handleBottomFontChange}
					/>
				</div>

				{#if bottomFontError}
					<div class="alert alert-warning">
						<span>{bottomFontError}</span>
					</div>
				{/if}

				<div class="divider my-1 sm:my-2"></div>


				<div class="form-control w-full">
					<label class="label">
						<span class="label-text py-2 sm:py-3 text-sm sm:text-base font-bold">Circle Color:</span>
					</label>
					<input
						class="h-8 sm:h-10 w-full cursor-pointer rounded border-2 border-gray-300"
						type="color"
						bind:value={circleColor}
						on:input={updateCircleColor}
					/>
				</div>

				<div class="form-control w-full">
					<label class="label">
						<span class="label-text py-2 sm:py-3 text-sm sm:text-base font-bold">Status Presets:</span>
					</label>
					<div class="flex flex-col gap-1 sm:gap-2">
						{#each presets as preset (preset.name)}
							<button
								class="btn w-full btn-xs sm:btn-sm text-xs sm:text-sm"
								style="background-color: {preset.color}; color: {preset.color === '#00CCFF'
									? '#000'
									: '#fff'}"
								on:click={() => applyPreset(preset)}
							>
								{preset.name}
							</button>
						{/each}
					</div>
				</div>

				<div class="mt-3 sm:mt-4 card-actions w-full">
					<button class="btn w-full shadow-sm btn-sm sm:btn-md btn-primary" on:click={exportSTL}>
						Export STL
					</button>
				</div>
			</div>
		</div>
	</div>

	<canvas class="block h-full w-full" bind:this={canvas}></canvas>
</div>
