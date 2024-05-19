<script lang="ts">
	import { FilesetResolver, HandLandmarker } from '@mediapipe/tasks-vision';
	import { onMount } from 'svelte';

	let handLandmarker: HandLandmarker;
	let hm = false;
	let enabled = false;
	let btn: HTMLButtonElement;
	let video: HTMLVideoElement;
	let handle: any;
	let hs: any = undefined;
	let world;
	let focusElem;

	let height: number, width: number;
	let d = 0,
		x = 400,
		y = 200,
		w = 192,
		h = 192;

	let isIn = (p1: any, p2: any) => {
		return p1.x > p2.x && p1.x < p2.x + p2.width && p1.y > p2.y && p1.y < p2.y + p2.height;
	};

	let format = (p: any) => {
		return { x: width - p.x * width, y: p.y * height };
	};

	const getDistance = (p1: any, p2: any) => {
		return Math.sqrt(Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2));
	};

	onMount(async () => {
		// get window size
		height = window.innerHeight;
		width = window.innerWidth;

		// track on window size change
		window.addEventListener('resize', () => {
			height = window.innerHeight;
			width = window.innerWidth;
		});

		const devices = await navigator.mediaDevices.enumerateDevices();
		const webcam = devices.find((d) => d.label === 'FaceTime HD Camera (2C0E:82E3)');
		navigator.mediaDevices
			.getUserMedia({
				video: {
					deviceId: webcam?.deviceId
				},
				audio: false
			})
			.then((stream) => {
				video.srcObject = stream;
				video.play();
			})
			.catch((err) => {
				console.error(`An error occurred: ${err}`);
			});

		const createHandLandmarker = async () => {
			const vision = await FilesetResolver.forVisionTasks(
				'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm'
			);
			handLandmarker = await HandLandmarker.createFromOptions(vision, {
				baseOptions: {
					modelAssetPath: `hand_landmarker.task`,
					delegate: 'GPU'
				},
				runningMode: 'VIDEO',
				numHands: 2
			});
			hm = true;
		};
		createHandLandmarker();

		handle = async () => {
			if (hm) {
				enabled = true;
				await handLandmarker.setOptions({ runningMode: 'VIDEO' });
				let lastVideoTime = -1;
				function renderLoop(): void {
					if (video.currentTime !== lastVideoTime) {
						const detections = handLandmarker.detectForVideo(video, video.currentTime);
						// processResults(detections);
						if (detections.landmarks.length > 0) {
							hs = detections.landmarks;
							world = detections.worldLandmarks;
							d = getDistance(hs[0][4], hs[0][8]);

							const checkPinch =
								(isIn(format(hs?.[0]?.[4]), { x: x, y: y, width: w, height: h }) ||
									isIn(format(hs?.[0]?.[8]), { x: x, y: y, width: w, height: h })) &&
								d < 0.04;

							if (checkPinch) {
								w = width - hs[0][4].x * width - 0.9 * x;
								h = hs[0][4].y * height - 0.9 * y;
							}

							// get element at position
							const elem = document.elementFromPoint(
								width - hs[0][8].x * width,
								hs[0][8].y * height
							);

							if (elem !== focusElem) {
								if (elem?.focus) {
									focusElem?.blur();
									focusElem = elem;
									focusElem.focus();
								}
							}
						} else hs = undefined;
						lastVideoTime = video.currentTime;
					}

					requestAnimationFrame(() => {
						renderLoop();
					});
				}
				renderLoop();
			}
		};
	});
</script>

<div
	class={`fixed h-[100svh] w-[100vw] top-0 left-0 flex justify-center items-end p-2 bg-neutral-950`}
>
	<div
		class="bg-red-500 fixed"
		style={`
            width: ${w}px; height: ${h}px;
            top: ${y}px; left: ${x}px;
        `}
	>
		<div class="relative w-full h-full flex justify-center items-center">
			<div class="flex flex-col gap-1">
				<div>{w.toFixed(2)} x {h.toFixed(2)}</div>
				<!-- <div>{x.toFixed(2)} x {y.toFixed(2)}</div> -->
			</div>
			<div
				class="absolute right-0 bottom-0 w-12 aspect-square"
				style={`background-color: ${
					hs
						? (isIn(format(hs?.[0]?.[4]), { x: x, y: y, width: w, height: h }) ||
								isIn(format(hs?.[0]?.[8]), { x: x, y: y, width: w, height: h })) &&
							d < 0.04
							? 'blue'
							: 'purple'
						: 'purple'
				};`}
			></div>
		</div>
	</div>
</div>

<div class="relative h-[100svh] w-[100vw] flex justify-center items-center">
	<button class="fixed w-20 aspect-video bg-blue-500 -translate-y-20 focus:bg-blue-200">asdf</button
	>
	<div class="hidden fixed top-0 left-0 flex flex-col p-1 text-white gap-1">
		<video playsinline bind:this={video} class="w-32 h-auto" style="transform: translateX(-1)">
			Video stream not available.
			<track kind="captions" />
		</video>
		<div class="flex flex-col gap-1">
			<div>{(width || 0).toFixed(2)} x {(height || 0).toFixed(2)}</div>
			<div>distance {d}</div>
		</div>
	</div>
	{#if hs}
		<div class=" h-full w-full">
			{#each hs as l, j}
				{#each l as p, i}
					<div
						class={`
                            w-6 text-neutral-100 flex justify-center items-center aspect-square text-sm rounded-full fixed border-neutral-600 border
                            ${d < 0.04 && j === 0 && [4, 8].indexOf(i) !== -1 ? 'bg-red-600' : 'bg-neutral-800'}
                        `}
						style={` left: ${width - p.x * width}px; top: ${p.y * height}px; `}
					>
						{i}
					</div>
				{/each}
			{/each}
		</div>
	{:else}
		<div class="text-neutral-100">{enabled ? 'No hand detected' : ''}</div>
	{/if}
</div>

<button
	bind:this={btn}
	on:click={handle}
	class={`${enabled ? 'bg-emerald-500' : 'bg-neutral-500'} text-sm text-white px-1 py-[0.15rem] rounded-md z-50 fixed bottom-0 right-0`}
	>{enabled ? 'Running' : 'Start Camera'}</button
>
