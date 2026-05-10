<script lang="ts">
	import Icon from "@iconify/svelte";
	import { onMount } from "svelte";

	interface Track {
		title: string;
		artist?: string;
		url: string;
		cover?: string;
	}

	const playlist: Track[] = [
		{
			title: "Call Me Now",
			artist: "神兔小P",
			url: "/music/call-me-now.mp3"
		},
		{
			title:"Terror Jr-3 Strikes",
			artist:'dark',
			url: "/music/domi.mp3"
		},
		{
			title:"恋人",
			artist:"李荣浩",
			"url":"/music/恋人.mp3"
		}
	];

	let currentIndex = $state(0);
	let isPlaying = $state(false);
	let currentTime = $state(0);
	let duration = $state(0);
	let volume = $state(0.7);
	let isExpanded = $state(false);
	let errorMsg = $state("");

	let audio: HTMLAudioElement;
	let isLoaded = $state(false);

	const currentTrack = $derived(playlist[currentIndex]);

	function togglePlay() {
		if (!audio) return;
		if (isPlaying) {
			isPlaying = false;
			audio.pause();
		} else {
			audio.play().then(() => {
				isPlaying = true;
			}).catch(e => {
				console.error("Playback error:", e);
				errorMsg = "播放失败，请检查网络";
			});
		}
	}

	function prevTrack() {
		currentIndex = (currentIndex - 1 + playlist.length) % playlist.length;
		loadAndPlay();
	}

	function nextTrack() {
		currentIndex = (currentIndex + 1) % playlist.length;
		loadAndPlay();
	}

	function loadTrack() {
		if (!audio) return;
		audio.src = currentTrack.url;
		audio.load();
	}

	function loadAndPlay() {
		if (!audio) return;
		audio.src = currentTrack.url;
		audio.load();
		audio.play().then(() => {
			isPlaying = true;
		}).catch(e => {
			console.error("Playback error:", e);
			errorMsg = "播放失败，请检查网络";
		});
	}

	function handleTimeUpdate() {
		if (!audio) return;
		currentTime = audio.currentTime;
	}

	function handleLoadedMetadata() {
		if (!audio) return;
		duration = audio.duration;
		isLoaded = true;
		console.log("Metadata loaded, duration:", duration);
	}

	function handleEnded() {
		nextTrack();
	}

	function handleError(e: Event) {
		console.error("Audio error:", e);
		errorMsg = "音频加载失败，请检查文件路径";
	}

	function handleProgressClick(e: MouseEvent) {
		if (!audio || !isLoaded) return;
		const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
		const percent = (e.clientX - rect.left) / rect.width;
		audio.currentTime = percent * duration;
	}

	function handleVolumeChange(e: Event) {
		const target = e.target as HTMLInputElement;
		volume = Number(target.value);
		if (audio) {
			audio.volume = volume;
		}
	}

	function formatTime(seconds: number): string {
		if (isNaN(seconds)) return "0:00";
		const m = Math.floor(seconds / 60);
		const s = Math.floor(seconds % 60);
		return `${m}:${s.toString().padStart(2, "0")}`;
	}

	onMount(() => {
		if (audio) {
			audio.volume = volume;
			loadTrack();
		}
	});
</script>

<audio
	bind:this={audio}
	onTimeUpdate={handleTimeUpdate}
	onLoadedMetadata={handleLoadedMetadata}
	onEnded={handleEnded}
	onError={handleError}
	preload="auto"
></audio>

<div class="w-full">
	{#if errorMsg}
		<div class="text-red-500 text-xs mb-2">{errorMsg}</div>
	{/if}

	<div class="flex items-center gap-3 mb-2">
		<button
			onclick={prevTrack}
			class="btn-plain rounded-lg h-9 w-9 flex items-center justify-center active:scale-90 transition"
			aria-label="上一首"
		>
			<Icon icon="material-symbols:skip-previous-rounded" class="text-xl"></Icon>
		</button>

		<button
			onclick={togglePlay}
			class="btn-regular rounded-full h-10 w-10 flex items-center justify-center active:scale-90 transition"
			aria-label={isPlaying ? "暂停" : "播放"}
		>
			{#if isPlaying}
				<Icon icon="material-symbols:pause-rounded" class="text-xl"></Icon>
			{:else}
				<Icon icon="material-symbols:play-arrow-rounded" class="text-xl"></Icon>
			{/if}
		</button>

		<button
			onclick={nextTrack}
			class="btn-plain rounded-lg h-9 w-9 flex items-center justify-center active:scale-90 transition"
			aria-label="下一首"
		>
			<Icon icon="material-symbols:skip-next-rounded" class="text-xl"></Icon>
		</button>

		<button
			onclick={() => isExpanded = !isExpanded}
			class="flex-1 min-w-0 text-left"
			aria-label="展开播放列表"
		>
			<div class="text-sm font-medium truncate transition text-[var(--primary)]">
				{currentTrack.title}
			</div>
			{#if currentTrack.artist}
				<div class="text-xs text-neutral-400 truncate">
					{currentTrack.artist}
				</div>
			{/if}
		</button>
	</div>

	<div
		onclick={handleProgressClick}
		class="h-1.5 bg-[var(--btn-regular-bg)] rounded-full cursor-pointer mb-2 overflow-hidden"
		role="slider"
		aria-label="播放进度"
		aria-valuenow={currentTime}
		aria-valuemin={0}
		aria-valuemax={duration}
	>
		<div
			class="h-full bg-[var(--primary)] rounded-full transition-all duration-100"
			style:width="{duration > 0 ? (currentTime / duration) * 100 : 0}%"
		></div>
	</div>

	<div class="flex items-center justify-between text-xs text-neutral-400 mb-2">
		<span>{formatTime(currentTime)}</span>
		<span>{formatTime(duration)}</span>
	</div>

	<div class="flex items-center gap-2">
		<Icon icon="material-symbols:volume-up-rounded" class="text-lg text-neutral-400"></Icon>
		<input
			type="range"
			min="0"
			max="1"
			step="0.01"
			value={volume}
			oninput={handleVolumeChange}
			class="w-full h-1 bg-[var(--btn-regular-bg)] rounded-full appearance-none cursor-pointer
				[&::-webkit-slider-thumb]:appearance-none [&::-webkit-slider-thumb]:w-3 [&::-webkit-slider-thumb]:h-3
				[&::-webkit-slider-thumb]:rounded-full [&::-webkit-slider-thumb]:bg-[var(--primary)]"
			aria-label="音量"
		/>
	</div>

	{#if isExpanded}
		<div class="mt-3 pt-3 border-t border-dashed border-neutral-200 dark:border-neutral-700">
			<div class="text-xs font-bold text-neutral-400 mb-2 uppercase tracking-wider">播放列表</div>
			{#each playlist as track, i}
				<button
					onclick={() => { currentIndex = i; loadAndPlay(); }}
					class="w-full text-left px-2 py-1.5 rounded-lg text-sm transition mb-0.5
						{i === currentIndex
							? 'bg-[var(--primary)] text-white'
							: 'hover:bg-[var(--btn-regular-bg)] text-neutral-600 dark:text-neutral-300'}"
				>
					<div class="truncate">{track.title}</div>
					{#if track.artist}
						<div class="text-xs opacity-60 truncate">{track.artist}</div>
					{/if}
				</button>
			{/each}
		</div>
	{/if}
</div>
