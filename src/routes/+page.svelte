<script lang="ts">
  import { onMount } from 'svelte';
  import { convertFileSrc } from '@tauri-apps/api/core';
  import { getCurrentWindow } from '@tauri-apps/api/window';
  import { readDir } from '@tauri-apps/plugin-fs';
  import { open } from '@tauri-apps/plugin-dialog';

  let filePath = $state('');
  let fileSrc = $state('');
  let fileName = $state('no file open');
  let isVideo = $state(false);
  let fileList: string[] = $state([]);
  let currentIndex = $state(0);

  let videoEl = $state<HTMLVideoElement | null>(null);
  let playing = $state(false);
  let muted = $state(false);
  let progress = $state(0);
  let currentTime = $state('0:00');
  let duration = $state('0:00');
  let showControls = $state(false);
  let hoverZone = $state('none');

  const imageExts = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'bmp', 'svg'];
  const videoExts = ['mp4', 'webm', 'mkv', 'avi', 'mov', 'wmv'];
  const allExts = [...imageExts, ...videoExts];

  function formatTime(seconds: number): string {
    const m = Math.floor(seconds / 60);
    const s = Math.floor(seconds % 60);
    return `${m}:${s.toString().padStart(2, '0')}`;
  }

  function updateProgress() {
    if (!videoEl) return;
    progress = (videoEl.currentTime / videoEl.duration) * 100;
    currentTime = formatTime(videoEl.currentTime);
    duration = formatTime(videoEl.duration);
    playing = !videoEl.paused;
  }

  function togglePlay() {
    if (!videoEl) return;
    videoEl.paused ? videoEl.play() : videoEl.pause();
    playing = !videoEl.paused;
  }

  function toggleMute() {
    if (!videoEl) return;
    videoEl.muted = !videoEl.muted;
    muted = videoEl.muted;
  }

  function startScrubbing(e: MouseEvent) {
    e.preventDefault();
    if (!videoEl) return;

    const bar = e.currentTarget as HTMLElement;
    const wasPlaying = !videoEl.paused;
    videoEl.pause();

    function scrubTo(clientX: number) {
      const rect = bar.getBoundingClientRect();
      const ratio = Math.max(0, Math.min(1, (clientX - rect.left) / rect.width));
      videoEl!.currentTime = ratio * videoEl!.duration;
      progress = ratio * 100;
      currentTime = formatTime(videoEl!.currentTime);
    }

    scrubTo(e.clientX);

    let lastScrub = 0;
    function onMouseMove(e: MouseEvent) {
      const now = Date.now();
      if (now - lastScrub < 60) return;
      lastScrub = now;
      scrubTo(e.clientX);
    }

    function onMouseUp() {
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('mouseup', onMouseUp);
      if (wasPlaying) videoEl!.play();
    }

    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('mouseup', onMouseUp);
  }

  function displayFile(path: string) {
    filePath = path;
    fileName = path.split('\\').pop() || path.split('/').pop() || path;
    const ext = path.split('.').pop()?.toLowerCase() || '';
    isVideo = videoExts.includes(ext);
    fileSrc = convertFileSrc(path);
  }

  async function loadFile(path: string) {
    displayFile(path);
    const sep = path.includes('\\') ? '\\' : '/';
    const folder = path.substring(0, path.lastIndexOf(sep));
    try {
      const entries = await readDir(folder);
      fileList = entries
        .filter(e => {
          const ext = e.name?.split('.').pop()?.toLowerCase() || '';
          return allExts.includes(ext);
        })
        .map(e => `${folder}${sep}${e.name}`)
        .sort();
      currentIndex = fileList.indexOf(path);
    } catch (err) {
      console.error('could not read folder:', err);
    }
  }

  function navigate(direction: number) {
    if (fileList.length === 0) return;
    currentIndex = (currentIndex + direction + fileList.length) % fileList.length;
    displayFile(fileList[currentIndex]);
  }

async function minimizeWindow() {
    await getCurrentWindow().minimize();
  }

  async function maximizeWindow() {
    await getCurrentWindow().toggleMaximize();
  }

  async function closeWindow() {
    await getCurrentWindow().close();
  }

async function startDrag(e: MouseEvent) {
    if ((e.target as HTMLElement).closest('button')) return;
    await getCurrentWindow().startDragging();
  }

  function handleKeydown(e: KeyboardEvent) {
    if (['ArrowRight', 'ArrowLeft', 'ArrowUp', 'ArrowDown', ' '].includes(e.key)) {
      e.preventDefault();
    }
    if (isVideo && videoEl && hoverZone === 'video') {
      if (e.key === ' ') togglePlay();
      if (e.key === 'ArrowRight') videoEl.currentTime = Math.min(videoEl.currentTime + 5, videoEl.duration);
      if (e.key === 'ArrowLeft') videoEl.currentTime = Math.max(videoEl.currentTime - 5, 0);
    } else {
      if (e.key === ' ' && isVideo && videoEl) togglePlay();
      if (e.key === 'ArrowRight') navigate(1);
      if (e.key === 'ArrowLeft') navigate(-1);
    }
  }

  async function openFileDialog() {
    const selected = await open({
      multiple: false,
      filters: [{
        name: 'Media',
        extensions: [...imageExts, ...videoExts]
      }]
    });
    if (selected) loadFile(selected as string);
  }

  onMount(() => {
    const initial = (window as any).__INITIAL_FILE__;
    if (initial) loadFile(initial);

    const appWindow = getCurrentWindow();
    appWindow.onDragDropEvent((event) => {
      if (event.payload.type === 'drop') {
        const paths = event.payload.paths;
        if (paths && paths.length > 0) loadFile(paths[0]);
      }
    });

    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  });
</script>

<main ondrop={(e) => e.preventDefault()} ondragover={(e) => e.preventDefault()}>
  <div class="topbar" onmousedown={startDrag} role="toolbar" tabindex="-1">
    <span class="app-name" data-tauri-drag-region>vyu</span>
    <span class="divider" data-tauri-drag-region>/</span>
    <span class="filename" data-tauri-drag-region>{fileName}</span>
    {#if fileSrc}
      <span class="divider" data-tauri-drag-region>/</span>
      <button class="folder-btn" onclick={openFileDialog} aria-label="open file">📁</button>
    {/if}
    <div class="window-controls" data-tauri-drag-region>
      <button class="wc-btn minimize" onclick={minimizeWindow} aria-label="minimize">−</button>
      <button class="wc-btn maximize" onclick={maximizeWindow} aria-label="maximize">▢</button>
      <button class="wc-btn close" onclick={closeWindow} aria-label="close">✕</button>
    </div>
  </div>

  <div class="content">
    <div class="sidebar left"
      onmouseenter={() => hoverZone = 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      <button class="nav-btn" onclick={() => navigate(-1)} aria-label="previous file">‹</button>
    </div>

    <div class="viewer"
      onmouseenter={() => hoverZone = isVideo ? 'video' : 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      {#if fileSrc && !isVideo}
        <img src={fileSrc} alt={fileName} />
      {:else if fileSrc && isVideo}
        <div class="video-wrapper"
          role="presentation"
          onmouseenter={() => hoverZone = 'video'}
          onmouseleave={() => hoverZone = 'none'}>
          <video
            bind:this={videoEl}
            src={fileSrc}
            autoplay
            ontimeupdate={updateProgress}
          >
            <track kind="captions" />
          </video>
          <div class="video-controls">
            <div class="progress-bar" 
              onmousedown={startScrubbing}
              role="slider"
              aria-label="video scrubber"
              aria-valuenow={progress}
              aria-valuemin={0}
              aria-valuemax={100}
              tabindex="0">
              <div class="progress-fill" style="width: {progress}%"></div>
              <div class="progress-diamond" style="left: {progress}%"></div>
            </div>
            <div class="controls-row">
              <button class="ctrl-btn" onclick={togglePlay}>{playing ? '⏸' : '▶'}</button>
              <button class="ctrl-btn" onclick={toggleMute}>{muted ? '🔇' : '🔊'}</button>
              <span class="time-display">{currentTime} / {duration}</span>
            </div>
          </div>
        </div>
      {:else}
        <button class="empty" onclick={openFileDialog}>
          <span class="empty-icon">+</span>
          <span class="empty-text">open a file</span>
        </button>
      {/if}
    </div>

    <div class="sidebar right"
      onmouseenter={() => hoverZone = 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      <button class="nav-btn" onclick={() => navigate(1)} aria-label="next file">›</button>
    </div>
  </div>

  <div class="bottombar">
    <span class="file-count">{fileList.length > 0 ? `${currentIndex + 1} / ${fileList.length}` : '—'}</span>
    <span class="file-info">{fileName}</span>
    <span class="zoom">100%</span>
  </div>
</main>

<style>
  main {
    width: 100vw;
    height: 100vh;
    background: #000000;
    margin: 0;
    padding: 0;
    overflow: hidden;
    display: flex;
    flex-direction: column;
  }

  .topbar {
    height: 36px;
    background: #111111;
    border-bottom: 0.5px solid #222222;
    display: flex;
    align-items: center;
    padding: 0 16px;
    gap: 8px;
    flex-shrink: 0;
    cursor: grab;
    user-select: none;
  }

  .app-name {
    font-size: 14px;
    color: #888888;
    font-family: Inter, sans-serif;
  }

  .divider {
    font-size: 14px;
    color: #444444;
    font-family: Inter, sans-serif;
  }

  .filename {
    font-size: 12px;
    color: #cccccc;
    font-family: Inter, sans-serif;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 300px;
  }

  .folder-btn {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 12px;
    padding: 2px 6px;
    border-radius: 4px;
    transition: background 0.2s;
    color: #666666;
  }

  .folder-btn:hover {
    background: #1a1a1a;
    color: #aaaaaa;
  }

  .window-controls {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .wc-btn {
    width: 28px;
    height: 28px;
    border-radius: 4px;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.2s, color 0.2s;
    color: #666666;
  }

  .wc-btn:hover {
    background: #1a1a1a;
    color: #cccccc;
  }

  .wc-btn.close:hover {
    background: #3a1a1a;
    color: #ff6666;
  }

  .wc-btn.maximize svg {
    opacity: 0.75;
  }

  .wc-btn.maximize:hover svg {
    opacity: 1;
  }

  .content {
    flex: 1;
    display: flex;
    flex-direction: row;
    overflow: hidden;
    background: #0a0a0a;
  }

  .sidebar {
    width: 48px;
    background: transparent;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
  }

  .nav-btn {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    background: #1a1a1a;
    border: 0.5px solid #2a2a2a;
    color: #444444;
    font-size: 22px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: color 0.2s, background 0.2s;
    line-height: 1;
  }

  .nav-btn:hover {
    color: #cccccc;
    background: #222222;
  }

  .viewer {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #0a0a0a;
    overflow: hidden;
    padding: 16px;
    box-sizing: border-box;
  }

  .viewer img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
    display: block;
  }

  .empty {
    background: none;
    border: 1px dashed #222222;
    border-radius: 8px;
    padding: 32px 48px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    transition: border-color 0.2s;
  }

  .empty:hover {
    border-color: #444444;
  }

  .empty-icon {
    font-size: 24px;
    color: #333333;
  }

  .empty-text {
    font-size: 13px;
    color: #333333;
    font-family: Inter, sans-serif;
  }

  .bottombar {
    height: 36px;
    background: #0d0d0d;
    border-top: 0.5px solid #1a1a1a;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 16px;
    flex-shrink: 0;
  }

  .file-count, .file-info, .zoom {
    font-size: 12px;
    color: #444444;
    font-family: Inter, sans-serif;
  }

  .video-wrapper {
    position: relative;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    max-width: 100%;
    max-height: 100%;
    line-height: 0;
    border-radius: 2px;
    outline: 4px solid transparent;
    transition: outline-color 0.5s;
  }

  .video-wrapper:hover {
    outline-color: #444444;
  }

  .video-wrapper video {
    display: block;
    max-width: calc(100vw - 96px - 32px);
    max-height: calc(100vh - 36px - 36px - 32px);
    width: auto;
    height: auto;
    object-fit: contain;
  }

  .video-wrapper:hover video {
    border-color: #444444;
  }

  .video-controls {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 8px 12px;
    background: linear-gradient(transparent, rgba(0,0,0,0.8));
    display: flex;
    flex-direction: column;
    gap: 6px;
    opacity: 0;
    transition: opacity 0.2s;
  }

  .video-wrapper:hover .video-controls {
    opacity: 1;
  }

  .progress-bar {
    width: 100%;
    height: 8px;
    background: #333333;
    border-radius: 2px;
    cursor: pointer;
    position: relative;
    border: none;
    padding: 0;
    overflow: visible;
  }

  .progress-fill {
    height: 100%;
    background: #ffffff;
    border-radius: 2px;
    pointer-events: none;
  }

  .progress-diamond {
    position: absolute;
    top: 50%;
    width: 14px;
    height: 14px;
    background: #ffffff;
    transform: translate(-50%, -50%) rotate(45deg);
    pointer-events: none;
    opacity: 0;
    transition: opacity 0.2s;
  }

  .video-wrapper:hover .progress-diamond {
    opacity: 1;
  }

  .controls-row {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .ctrl-btn {
    background: none;
    border: none;
    color: #cccccc;
    cursor: pointer;
    font-size: 20px;
    padding: 2px 4px;
    font-family: Inter, sans-serif;
  }

  .ctrl-btn:hover {
    color: #ffffff;
  }

  .time-display {
    font-size: 15px;
    color: #888888;
    font-family: Inter, sans-serif;
    margin-left: auto;
  }
</style>