<script lang="ts">
import Icon from "@iconify/svelte";
import { onMount } from "svelte";

let isRunning = $state(false);
let { ...props }: { [key: string]: unknown } = $props();
let totalTime = $state(0); // in ms
let laps = $state<{ id: number; time: number; split: number }[]>([]);

let intervalId: any = null;
let startTime = 0;
let elapsedBefore = 0;

// Current dynamic rotation angle for visual timer ring
let rotationAngle = $state(0);
let rotationInterval: any = null;

function formatTime(ms: number) {
	const minutes = Math.floor(ms / 60000);
	const seconds = Math.floor((ms % 60000) / 1000);
	const centiseconds = Math.floor((ms % 1000) / 10);
	return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}.${centiseconds.toString().padStart(2, '0')}`;
}

function start() {
	if (isRunning) return;
	isRunning = true;
	startTime = performance.now();
	intervalId = setInterval(() => {
		totalTime = elapsedBefore + (performance.now() - startTime);
	}, 10);

	// Rotate ring visual effect
	rotationInterval = setInterval(() => {
		rotationAngle = rotationAngle + 3;
	}, 16); // ~60fps
}

function pause() {
	if (!isRunning) return;
	isRunning = false;
	elapsedBefore = totalTime;
	clearInterval(intervalId);
	clearInterval(rotationInterval);
}

function reset() {
	pause();
	totalTime = 0;
	elapsedBefore = 0;
	laps = [];
	rotationAngle = 0;
}

function addLap() {
	if (!isRunning && totalTime === 0) return;
	const currentLapTime = laps.length === 0 ? totalTime : totalTime - laps[0].split;
	laps = [
		{
			id: laps.length + 1,
			time: currentLapTime,
			split: totalTime
		},
		...laps
	];
}

onMount(() => {
	return () => {
		clearInterval(intervalId);
		clearInterval(rotationInterval);
	};
});
</script>

<div class="flex flex-col items-center gap-6 max-w-md w-full mx-auto p-6 card-base backdrop-blur-md">
    <!-- Visual Timer Ring -->
    <div class="relative w-56 h-56 flex items-center justify-center rounded-full bg-black/5 dark:bg-white/5 border border-black/10 dark:border-white/10 shadow-inner">
        <!-- Spinning indicator ring -->
        <svg class="absolute top-0 left-0 w-full h-full transform -rotate-90 pointer-events-none" viewBox="0 0 100 100">
            <circle cx="50" cy="50" r="46" stroke="currentColor" stroke-width="2" class="text-neutral-200/20 dark:text-neutral-700/20" fill="transparent" />
            <circle cx="50" cy="50" r="46" stroke="var(--primary)" stroke-width="2" 
                    stroke-dasharray="290" 
                    stroke-dashoffset={290 - (totalTime % 60000 / 60000) * 290}
                    class="transition-all duration-75" 
                    fill="transparent" 
                    stroke-linecap="round" />
        </svg>

        <!-- Dynamic dot sweeping when running -->
        <div class="absolute w-full h-full pointer-events-none" style="transform: rotate({rotationAngle}deg);">
            <div class="absolute top-2 left-1/2 -translate-x-1/2 w-2.5 h-2.5 rounded-full bg-[var(--primary)] shadow-[0_0_8px_var(--primary)]"></div>
        </div>

        <!-- Timer Text Display -->
        <div class="flex flex-col items-center gap-1.5 z-10 select-none">
            <span class="text-xs uppercase tracking-wider text-neutral-400 dark:text-neutral-500 font-semibold">在线秒表</span>
            <span class="font-mono text-4xl font-bold tracking-tight text-neutral-800 dark:text-neutral-100 tabular-nums">
                {formatTime(totalTime)}
            </span>
            <span class="text-[10px] text-neutral-400 dark:text-neutral-500">
                {#if isRunning}
                    🟢 运行中
                {:else if totalTime > 0}
                    🟡 已暂停
                {:else}
                    ⚪ 准备就绪
                {/if}
            </span>
        </div>
    </div>

    <!-- Controls Button Grid -->
    <div class="grid grid-cols-3 gap-4 w-full">
        <!-- Lap/Reset Button -->
        {#if isRunning}
            <button onclick={addLap} class="btn-regular rounded-xl h-11 font-bold text-sm flex items-center justify-center gap-1.5 active:scale-95 transition-all">
                <Icon icon="material-symbols:flag-outline-rounded" class="text-lg" />
                计圈
            </button>
        {:else}
            <button onclick={reset} disabled={totalTime === 0} class="btn-regular rounded-xl h-11 font-bold text-sm flex items-center justify-center gap-1.5 active:scale-95 transition-all disabled:opacity-40 disabled:pointer-events-none">
                <Icon icon="material-symbols:restart-alt-rounded" class="text-lg" />
                复位
            </button>
        {/if}

        <!-- Play/Pause Button -->
        {#if !isRunning}
            <button onclick={start} class="col-span-2 btn-regular rounded-xl h-11 font-bold text-sm flex items-center justify-center gap-1.5 bg-[var(--primary)] hover:bg-[var(--primary-hover)] text-white active:scale-95 transition-all">
                <Icon icon="material-symbols:play-arrow-rounded" class="text-xl" />
                开始计时
            </button>
        {:else}
            <button onclick={pause} class="col-span-2 btn-regular rounded-xl h-11 font-bold text-sm flex items-center justify-center gap-1.5 bg-yellow-500 hover:bg-yellow-600 text-white active:scale-95 transition-all">
                <Icon icon="material-symbols:pause-rounded" class="text-xl" />
                暂停计时
            </button>
        {/if}
    </div>

    <!-- Lap list area -->
    {#if laps.length > 0}
        <div class="w-full flex flex-col gap-2 max-h-48 overflow-y-auto pr-1 border-t border-black/5 dark:border-white/5 pt-4">
            <span class="text-xs font-semibold text-neutral-400 dark:text-neutral-500 mb-1 pl-1">计圈历史记录</span>
            {#each laps as lap (lap.id)}
                <div class="flex items-center justify-between py-1.5 px-3 rounded-lg bg-black/5 dark:bg-white/5 border border-black/5 dark:border-white/5 text-xs text-neutral-600 dark:text-neutral-400 transition hover:border-black/10 dark:hover:border-white/10">
                    <div class="flex items-center gap-2">
                        <span class="text-[var(--primary)] font-bold">#{lap.id}</span>
                        <span class="text-[10px] text-neutral-400 dark:text-neutral-500">单圈时间</span>
                    </div>
                    <div class="flex items-center gap-4">
                        <span class="font-mono">{formatTime(lap.time)}</span>
                        <span class="font-mono text-[10px] text-neutral-400 dark:text-neutral-500">累计 {formatTime(lap.split)}</span>
                    </div>
                </div>
            {/each}
        </div>
    {/if}
</div>
