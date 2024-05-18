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

	onMount(async () => {
		navigator.mediaDevices
			.getUserMedia({ video: true, audio: false })
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
						if (detections.landmarks.length > 0) hs = detections.landmarks;
						else hs = undefined;
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

<div class="flex flex-col">
	<video playsinline bind:this={video} class="w-0 h-0">
		Video stream not available.
		<track kind="captions" />
	</video>
</div>

<div class="relative h-[100svh] w-[100vw] flex justify-center items-center bg-neutral-950">
	{#if hs}
		<div class=" h-full w-full" style="transform: scaleX(-1)">
			{#each hs as l}
				{#each l as p, i}
					<div
						class="w-6 text-neutral-100 flex justify-center items-center aspect-square text-sm rounded-full fixed border-neutral-600 border bg-neutral-800"
						style={`
            left: ${p.x * 100}%;
            top: ${p.y * 100}%;
            transform: scaleX(-1);
        `}
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

<div class={`fixed h-[100svh] w-[100vw] top-0 left-0 flex justify-center items-end p-2`}>
	<button
		bind:this={btn}
		on:click={handle}
		class={`${enabled ? 'bg-emerald-500' : 'bg-neutral-500'} text-sm text-white px-1 py-[0.15rem] rounded-md`}
		>{enabled ? 'Running' : 'Start Camera'}</button
	>
</div>
