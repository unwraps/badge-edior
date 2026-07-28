<script lang="ts">
	import { onMount } from 'svelte';
	import Icon from '@iconify/svelte';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
	import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
	import { STLExporter } from 'three/examples/jsm/exporters/STLExporter.js';
	import { STLLoader } from 'three/examples/jsm/loaders/STLLoader.js';

	let canvas: HTMLCanvasElement;
	let topText = $state('TOP TEXT');
	let bottomText = $state('BOTTOM TEXT');
	let circleColor = $state('#FF0000'); // Red by default
	let topFontError: string | null = $state(null);
	let bottomFontError: string | null = $state(null);
	let showGrid = $state(true);
	let vrchatMode = $state(true);
	let offsetX = $state(23);
	let offsetY = $state(-11.5);
	let offsetZ = $state(-2);
	let exporting = $state(false);
	let topFileName = $state('');
	let bottomFileName = $state('');
	let sidebarOpen = $state(true);
	let panelCollapsed = $state(false);
	let baseScale = $state(1.0);
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
	let gridHelper: THREE.GridHelper | null = null;
	let vrchatBaseGroup: THREE.Group | null = null;
	let resizeObserver: ResizeObserver | null = null;
	let loadedBaseObject: THREE.Object3D | null = null;
	let baseInitialScaleFactor = 1;
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
		// Apply default badge position offset
		updateBadgePosition();
		// Load VRChat base if enabled by default
		if (vrchatMode) {
			loadVrchatBase();
		}
		// Load default font for both top and bottom
		fontLoader.load('/fonts/helvetiker_regular.typeface.json', (font) => {
			topFont = font;
			bottomFont = font;
			updateTopText();
			updateBottomText();
		});

		return () => {
			resizeObserver?.disconnect();
			renderer.dispose();
		};
	});

	function getCanvasSize() {
		const parent = canvas.parentElement;
		if (!parent) return { w: 0, h: 0 };
		return { w: parent.clientWidth, h: parent.clientHeight };
	}

	function initScene() {
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0xefefef);

		const { w: initW, h: initH } = getCanvasSize();
		camera = new THREE.PerspectiveCamera(50, initW / initH || 1, 0.1, 1000);
		camera.position.set(0, 0, 150);

		renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
		renderer.setSize(initW, initH);

		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;

		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);

		const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
		dirLight.position.set(50, 50, 100);
		scene.add(dirLight);

		// Initialize circle material with the current color
		circleMaterial = new THREE.MeshStandardMaterial({ color: circleColor });

		// Grid helper (always visible, positioned below the model)
		gridHelper = new THREE.GridHelper(400, 40, 0xcccccc, 0xdddddd);
		gridHelper.position.y = -30;
		gridHelper.visible = true;
		gridHelper.material.opacity = 0.5;
		gridHelper.material.color.setStyle('#cccccc');
		scene.add(gridHelper);

		// VRChat base group (hidden by default)
		vrchatBaseGroup = new THREE.Group();
		scene.add(vrchatBaseGroup);

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

		// Force an initial resize after first paint to ensure correct dimensions
		requestAnimationFrame(() => requestAnimationFrame(() => onWindowResize()));

		// Watch the parent container for size changes (sidebar collapse, window resize, etc.)
		const container = canvas.parentElement!;
		resizeObserver = new ResizeObserver(() => {
			requestAnimationFrame(() => onWindowResize());
		});
		resizeObserver.observe(container);
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
		if (!camera || !renderer || !canvas) return;
		const { w, h } = getCanvasSize();
		if (w === 0 || h === 0) return;
		camera.aspect = w / h;
		camera.updateProjectionMatrix();
		renderer.setSize(w, h);
	}

	function animate() {
		requestAnimationFrame(animate);
		controls.update();
		renderer.render(scene, camera);
	}

	function exportSTL() {
		exporting = true;
		// Use setTimeout to let the UI update before the potentially heavy export
		setTimeout(() => {
			try {
				// Temporarily reset badge position so export is centered at origin
				const prevPos = badgeGroup.position.clone();
				badgeGroup.position.set(0, 0, 0);

				const exporter = new STLExporter();
				const stlString = exporter.parse(badgeGroup);

				// Restore position
				badgeGroup.position.copy(prevPos);

				const blob = new Blob([stlString], { type: 'text/plain' });
				const url = URL.createObjectURL(blob);
				const link = document.createElement('a');
				link.style.display = 'none';
				link.href = url;
				link.download = 'badge.stl';
				document.body.appendChild(link);
				link.click();
				document.body.removeChild(link);
				URL.revokeObjectURL(url);
			} catch (e) {
				console.error('Export failed:', e);
			} finally {
				exporting = false;
			}
		}, 50);
	}

	function toggleGrid() {
		showGrid = !showGrid;
		if (gridHelper) {
			gridHelper.visible = showGrid;
		}
	}

	function loadVrchatBase() {
		if (!vrchatBaseGroup) return;
		baseScale = 1.0;
		const stlLoader = new STLLoader();
		fetch('/models/vrchat-badge-base.stl')
			.then((res) => res.arrayBuffer())
			.then((data) => {
				const geometry = stlLoader.parse(data);
				geometry.computeBoundingBox();
				const box = geometry.boundingBox!;
				const centerX = (box.max.x + box.min.x) / 2;
				const centerY = (box.max.y + box.min.y) / 2;
				const centerZ = (box.max.z + box.min.z) / 2;
				geometry.translate(-centerX, -centerY, -centerZ);

				const baseMesh = new THREE.Mesh(
					geometry,
					new THREE.MeshStandardMaterial({
						color: 0x4488ff,
						transparent: true,
						opacity: 0.4,
						side: THREE.DoubleSide
					})
				);
				baseInitialScaleFactor = 1;
				if (vrchatBaseGroup) {
					vrchatBaseGroup.add(baseMesh);
				}
				loadedBaseObject = baseMesh;
			})
			.catch((err) => console.error('Failed to load VRChat base:', err));
	}

	function clearVrchatBase() {
		if (!vrchatBaseGroup) return;
		while (vrchatBaseGroup.children.length > 0) {
			const child = vrchatBaseGroup.children[0];
			vrchatBaseGroup.remove(child);
			if (child instanceof THREE.Mesh) {
				child.geometry?.dispose();
			}
		}
		loadedBaseObject = null;
	}

	function toggleVrchatMode() {
		vrchatMode = !vrchatMode;
		if (vrchatMode) {
			loadVrchatBase();
		} else {
			clearVrchatBase();
		}
		// Reset offsets when toggling mode
		offsetX = 23;
		offsetY = -11.5;
		offsetZ = -2;
		updateBadgePosition();
	}

	function updateBaseScale() {
		if (loadedBaseObject) {
			const s = baseInitialScaleFactor * baseScale;
			loadedBaseObject.scale.set(s, s, s);
		}
	}

	function updateBadgePosition() {
		if (badgeGroup) {
			badgeGroup.position.set(offsetX, offsetY, offsetZ);
		}
	}

	function setPanelCollapsed(collapsed: boolean) {
		panelCollapsed = collapsed;
	}

	// React to panel collapse — ResizeObserver on the container also handles this,
	// but $effect ensures we catch any edge case where the observer doesn't fire
	$effect(() => {
		panelCollapsed;
		onWindowResize();
	});

	function handleTopFontChange(e: Event) {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			topFontError = null;
			topFileName = file.name;
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
			bottomFileName = file.name;
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

<div class="flex h-screen w-screen overflow-hidden bg-base-200">
	<!-- ===== MOBILE: persistent bottom handle + drawer (sm:hidden) ===== -->
	<!-- When drawer is open, show backdrop -->
	{#if sidebarOpen}
		<button
			class="fixed inset-0 z-30 cursor-default bg-black/40 sm:hidden"
			onclick={() => (sidebarOpen = false)}
			aria-label="Close controls"
		></button>
	{/if}

	<!-- Drawer panel (slides up) -->
	<div
		class="fixed bottom-0 left-0 right-0 z-40 flex flex-col sm:hidden transition-transform duration-300 ease-out
			{sidebarOpen ? 'translate-y-0' : 'translate-y-[calc(100%-44px)]'}">
		<div class="flex max-h-[85vh] flex-col overflow-hidden rounded-t-2xl bg-base-100 shadow-2xl">
			<!-- Persistent handle / drag bar — always visible -->
			<button
				class="flex w-full flex-col items-center gap-0.5 pt-2 pb-2 focus:outline-none active:bg-base-200/50"
				onclick={() => (sidebarOpen = !sidebarOpen)}
				aria-label={sidebarOpen ? 'Close controls' : 'Open controls'}
			>
				<div class="h-1.5 w-10 rounded-full bg-base-300"></div>
				<span class="text-[10px] font-semibold text-base-content/50 uppercase tracking-wider">
					{sidebarOpen ? 'Close' : 'Controls'}
				</span>
			</button>

			<!-- Close button row (only when open) -->
			{#if sidebarOpen}
				<div class="flex items-center justify-between px-4 pb-1">
					<h3 class="text-sm font-bold">Badge Editor</h3>
					<button
						class="btn btn-ghost btn-xs btn-square"
						onclick={() => (sidebarOpen = false)}
						aria-label="Close"
					>
						<Icon icon="mdi:close" class="size-5" />
					</button>
				</div>
				<!-- Scrollable content -->
				<div class="flex-1 overflow-y-auto px-4 pb-6">
					{@render panelContent()}
				</div>
			{/if}
		</div>
	</div>

	<!-- ===== DESKTOP / TABLET: static sidebar ===== -->
	<aside
		class="hidden flex-col border-r border-base-300 bg-base-100 shadow-sm transition-all duration-200 sm:flex"
		class:w-80={!panelCollapsed}
		class:w-14={panelCollapsed}
	>
		<!-- Header row -->
		<div class="flex bg-primary text-primary-content items-center gap-2 border-b border-base-300 p-3 {panelCollapsed ? 'justify-center' : ''}">
			{#if !panelCollapsed}
				<div class="flex size-8 items-center justify-center bg-primary text-primary-content text-xs font-bold">
					<Icon icon="mdi:shield-check" class="size-4" />
				</div>
				<div class="flex-1 min-w-0">
					<h2 class="card-title text-sm">Badge Editor</h2>
					<p class="text-[10px] truncate">Very nice badge right??</p>
				</div>
				<button
					class="btn btn-ghost btn-xs btn-square shrink-0"
					onclick={() => setPanelCollapsed(true)}
					aria-label="Collapse sidebar"
					title="Collapse"
				>
					<Icon icon="mdi:chevron-left" class="size-4" />
				</button>
			{:else}
				<button
					class="btn btn-ghost btn-xs btn-square"
					onclick={() => setPanelCollapsed(false)}
					aria-label="Expand sidebar"
					title="Expand"
				>
					<Icon icon="mdi:chevron-right" class="size-5" />
				</button>
			{/if}
		</div>

		{#if !panelCollapsed}
			<!-- Scrollable content -->
			<div class="flex-1 overflow-y-auto">
				<div class="flex flex-col gap-1 p-3">
					{@render panelContent()}
				</div>
			</div>
		{/if}
	</aside>

	<!-- ===== CANVAS AREA ===== -->
	<main class="relative flex-1 min-w-0">
		<canvas class="block size-full" bind:this={canvas}></canvas>
	</main>
</div>

<!-- ===== SHARED PANEL CONTENT ===== -->
{#snippet panelContent()}
	<div class="flex flex-col gap-2">
		<!-- === TEXT SECTION === -->
		<div class="collapse collapse-arrow">
			<input type="checkbox" checked />
			<div class="collapse-title flex items-center gap-2 px-0 py-2 text-sm font-bold">
				<Icon icon="mdi:format-text" class="size-4 shrink-0" />
				Text
			</div>
			<div class="collapse-content px-0 pb-2">
				<div class="flex flex-col gap-2">
					<div class="form-control w-full">
						<label for="top-text" class="label py-1">
							<span class="label-text text-xs font-semibold">Top Plate</span>
						</label>
						<input
							id="top-text"
							class="input-bordered input input-sm w-full"
							type="text"
							placeholder="TOP TEXT"
							bind:value={topText}
							oninput={updateTopText}
						/>
					</div>
					<div class="form-control w-full">
						<label for="bottom-text" class="label py-1">
							<span class="label-text text-xs font-semibold">Bottom Plate</span>
						</label>
						<input
							id="bottom-text"
							class="input-bordered input input-sm w-full"
							type="text"
							placeholder="BOTTOM TEXT"
							bind:value={bottomText}
							oninput={updateBottomText}
						/>
					</div>
				</div>
			</div>
		</div>

		<div class="divider my-0"></div>

		<!-- === FONTS SECTION === -->
		<div class="collapse collapse-arrow">
			<input type="checkbox" />
			<div class="collapse-title flex items-center gap-2 px-0 py-2 text-sm font-bold">
				<Icon icon="mdi:format-font" class="size-4 shrink-0" />
				Fonts
			</div>
			<div class="collapse-content px-0 pb-2">
				<div class="flex flex-col gap-2">
					<div class="form-control w-full">
						<label for="top-font" class="label py-1">
							<span class="label-text text-xs font-semibold">Top Font</span>
						</label>
						<input
							id="top-font"
							class="file-input-bordered file-input file-input-sm w-full"
							type="file"
							accept=".json"
							onchange={handleTopFontChange}
						/>
						{#if topFileName}
							<span class="mt-0.5 text-xs text-base-content/60">{topFileName}</span>
						{/if}
					</div>
					{#if topFontError}
						<div class="alert alert-warning py-1 text-xs">
							<span>{topFontError}</span>
						</div>
					{/if}
					<div class="form-control w-full">
						<label for="bottom-font" class="label py-1">
							<span class="label-text text-xs font-semibold">Bottom Font</span>
						</label>
						<input
							id="bottom-font"
							class="file-input-bordered file-input file-input-sm w-full"
							type="file"
							accept=".json"
							onchange={handleBottomFontChange}
						/>
						{#if bottomFileName}
							<span class="mt-0.5 text-xs text-base-content/60">{bottomFileName}</span>
						{/if}
					</div>
					{#if bottomFontError}
						<div class="alert alert-warning py-1 text-xs">
							<span>{bottomFontError}</span>
						</div>
					{/if}
				</div>
			</div>
		</div>

		<div class="divider my-0"></div>

		<!-- === APPEARANCE SECTION === -->
		<div class="collapse collapse-arrow">
			<input type="checkbox" />
			<div class="collapse-title flex items-center gap-2 px-0 py-2 text-sm font-bold">
				<Icon icon="mdi:palette" class="size-4 shrink-0" />
				Appearance
			</div>
			<div class="collapse-content px-0 pb-2">
				<div class="flex flex-col gap-2">
					<div class="form-control w-full">
						<label for="circle-color" class="label py-1">
							<span class="label-text text-xs font-semibold">Status Circle Color</span>
						</label>
						<div class="flex items-center gap-3">
							<div
								class="size-8 shrink-0 rounded border-2 border-base-300"
								style="background-color: {circleColor}"
							></div>
							<input
								id="circle-color"
								class="h-8 w-full cursor-pointer rounded border-2 border-base-300 bg-transparent"
								type="color"
								bind:value={circleColor}
								oninput={updateCircleColor}
							/>
						</div>
					</div>

					<div class="form-control w-full">
						<span class="label py-1">
							<span class="label-text text-xs font-semibold">Quick Presets</span>
						</span>
						<div class="grid grid-cols-2 gap-1.5">
							{#each presets as preset (preset.name)}
								<button
									class="btn btn-xs flex items-center gap-1.5 border-0 text-xs"
									style="background-color: {preset.color}22; color: {preset.color}; border: 1px solid {preset.color}44;"
									onclick={() => applyPreset(preset)}
								>
									<span
										class="size-2.5 shrink-0 rounded-full"
										style="background-color: {preset.color}"
									></span>
									{preset.name}
								</button>
							{/each}
						</div>
					</div>
				</div>
			</div>
		</div>

		<div class="divider my-0"></div>

		<!-- === SCENE SECTION === -->
		<div class="collapse collapse-arrow">
			<input type="checkbox" />
			<div class="collapse-title flex items-center gap-2 px-0 py-2 text-sm font-bold">
				<Icon icon="mdi:cube-outline" class="size-4 shrink-0" />
				Scene
			</div>
			<div class="collapse-content px-0 pb-2">
				<div class="flex flex-col gap-2">
					<div class="form-control w-full">
						<label class="label cursor-pointer gap-3 py-1.5">
							<span class="label-text text-xs">VRChat Base Mode</span>
							<input
								type="checkbox"
								class="toggle toggle-sm toggle-primary"
								checked={vrchatMode}
								onchange={toggleVrchatMode}
							/>
						</label>
					</div>

					{#if vrchatMode}
						<div class="mt-1 flex flex-col gap-2 rounded-lg bg-base-200/60 p-2.5">
							<div class="form-control w-full">
								<div class="flex items-center justify-between">
									<span class="label-text text-xs font-semibold">Base Scale</span>
									<span class="text-xs font-mono text-base-content/70">{baseScale.toFixed(2)}x</span>
								</div>
								<input
									type="range"
									class="range range-xs mt-0.5"
									min="0.1"
									max="3.0"
									step="0.05"
									bind:value={baseScale}
									oninput={updateBaseScale}
								/>
							</div>
							<div class="divider my-0.5"></div>
							<span class="text-xs font-bold uppercase tracking-wider text-base-content/70">Badge Offset</span>
							<div class="form-control w-full">
								<div class="flex items-center justify-between">
									<label for="offset-x" class="label py-0.5">
										<span class="label-text text-xs font-mono">X</span>
									</label>
									<span class="text-xs font-mono text-base-content/70">{offsetX.toFixed(1)}</span>
								</div>
								<input
									id="offset-x"
									type="range"
									class="range range-xs mt-0.5"
									min="-100"
									max="100"
									step="0.5"
									bind:value={offsetX}
									oninput={updateBadgePosition}
								/>
							</div>
							<div class="form-control w-full">
								<div class="flex items-center justify-between">
									<label for="offset-y" class="label py-0.5">
										<span class="label-text text-xs font-mono">Y</span>
									</label>
									<span class="text-xs font-mono text-base-content/70">{offsetY.toFixed(1)}</span>
								</div>
								<input
									id="offset-y"
									type="range"
									class="range range-xs mt-0.5"
									min="-100"
									max="100"
									step="0.5"
									bind:value={offsetY}
									oninput={updateBadgePosition}
								/>
							</div>
							<div class="form-control w-full">
								<div class="flex items-center justify-between">
									<label for="offset-z" class="label py-0.5">
										<span class="label-text text-xs font-mono">Z</span>
									</label>
									<span class="text-xs font-mono text-base-content/70">{offsetZ.toFixed(1)}</span>
								</div>
								<input
									id="offset-z"
									type="range"
									class="range range-xs mt-0.5"
									min="-100"
									max="100"
									step="0.5"
									bind:value={offsetZ}
									oninput={updateBadgePosition}
								/>
							</div>
						</div>
					{/if}
				</div>
			</div>
		</div>

		<div class="divider my-0"></div>

		<!-- Export -->
		<div class="flex flex-col gap-2 pt-1">
			<button
				class="btn w-full shadow-sm btn-sm btn-primary"
				onclick={exportSTL}
				disabled={exporting}
			>
				{#if exporting}
					<span class="loading loading-spinner loading-xs"></span>
					Exporting...
				{:else}
					<Icon icon="mdi:download" class="size-4" />
					Export STL
				{/if}
			</button>
			<p class="text-center text-[10px] text-base-content/40">
				Drag to rotate &bull; Scroll to zoom &bull; Right-click to pan
			</p>
			<p class="text-center text-[9px] text-base-content/30">
				Made by <a href="https://github.com/unwrapshell" target="_blank" rel="noopener noreferrer" class="hover:text-base-content/60 transition-colors">@unwrapshell</a>
			</p>
		</div>
	</div>
{/snippet}
