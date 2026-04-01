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

  function seekVideo(e: MouseEvent) {
    if (!videoEl) return;
    const bar = e.currentTarget as HTMLElement;
    const ratio = e.offsetX / bar.offsetWidth;
    videoEl.currentTime = ratio * videoEl.duration;
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
  <div class="topbar">
    <span class="app-name">vyu</span>
    <span class="divider">/</span>
    <span class="filename">{fileName}</span>
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
            <button class="progress-bar" onclick={seekVideo} aria-label="seek video">
              <div class="progress-fill" style="width: {progress}%"></div>
            </button>
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
  }

  .app-name {
    font-size: 12px;
    color: #888888;
    font-family: Inter, sans-serif;
  }

  .divider {
    font-size: 12px;
    color: #444444;
    font-family: Inter, sans-serif;
  }

  .filename {
    font-size: 12px;
    color: #cccccc;
    font-family: Inter, sans-serif;
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
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: #1a1a1a;
    border: 0.5px solid #2a2a2a;
    color: #444444;
    font-size: 18px;
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
    font-size: 11px;
    color: #444444;
    font-family: Inter, sans-serif;
  }

 .video-wrapper {
    position: relative;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border: 4px solid transparent;
    border-radius: 2px;
    transition: border-color 0.2s;
  }

  .video-wrapper:hover {
    border-color: #444444;
  }

  .video-wrapper video {
    max-width: 100%;
    max-height: 100%;
    display: block;
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
    height: 3px;
    background: #333333;
    border-radius: 2px;
    cursor: pointer;
    position: relative;
    border: none;
    padding: 0;
  }

  .progress-fill {
    height: 100%;
    background: #ffffff;
    border-radius: 2px;
    pointer-events: none;
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
    font-size: 13px;
    padding: 2px 4px;
    font-family: Inter, sans-serif;
  }

  .ctrl-btn:hover {
    color: #ffffff;
  }

  .time-display {
    font-size: 11px;
    color: #888888;
    font-family: Inter, sans-serif;
    margin-left: auto;
  }
</style>