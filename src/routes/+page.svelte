<script lang="ts">
  import { onMount } from 'svelte';
  import { convertFileSrc } from '@tauri-apps/api/core';
  import { getCurrentWindow } from '@tauri-apps/api/window';
  import { readDir, stat } from '@tauri-apps/plugin-fs';
  import { open } from '@tauri-apps/plugin-dialog';

  // file state
  let filePath = $state('');
  let fileSrc = $state('');
  let fileName = $state('no file open');
  let isVideo = $state(false);
  let fileList: string[] = $state([]);
  let currentIndex = $state(0);
  let fileSize = $state('');
  let fileDimensions = $state('');
  let fileInfoLoading = $state(false);

  // video state
  let videoEl = $state<HTMLVideoElement | null>(null);
  let playing = $state(false);
  let muted = $state(false);
  let progress = $state(0);
  let currentTime = $state('0:00');
  let duration = $state('0:00');

  // volume state
  let volume = $state(1);
  let volumeHovered = $state(false);
  const VOLUME_SEGMENTS = 8;

  // ui state
  let hoverZone = $state('none');
  let isFullscreen = $state(false);
  let fsControlsVisible = $state(true);
  let fsHideTimer: ReturnType<typeof setTimeout> | null = null;
  let zoomLevel = $state(100);

  // constants
  const imageExts = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'bmp', 'svg'];
  const videoExts = ['mp4', 'webm', 'mkv', 'avi', 'mov', 'wmv'];
  const allExts = [...imageExts, ...videoExts];

  // helpers
  function formatTime(seconds: number): string {
    if (!seconds || isNaN(seconds)) return '0:00';
    const m = Math.floor(seconds / 60);
    const s = Math.floor(seconds % 60);
    return `${m}:${s.toString().padStart(2, '0')}`;
  }

  function formatFileSize(bytes: number): string {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
  }

  // video controls
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
    muted = !muted;
    videoEl.muted = muted;
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

  // volume controls
  function setVolume(val: number) {
    volume = Math.max(0, Math.min(1, val));
    if (videoEl) {
      videoEl.volume = volume;
      muted = volume === 0;
      videoEl.muted = muted;
    }
    saveVolume();
  }

  function handleVolumeScroll(e: WheelEvent) {
    e.preventDefault();
    const delta = e.deltaY > 0 ? -0.125 : 0.125;
    setVolume(volume + delta);
  }

  function startVolumeDrag(e: MouseEvent) {
    e.preventDefault();
    const bar = e.currentTarget as HTMLElement;
    const diamonds = bar.querySelectorAll('.volume-diamond');

    function dragTo(clientX: number) {
      const first = diamonds[0].getBoundingClientRect();
      const last = diamonds[diamonds.length - 1].getBoundingClientRect();
      const ratio = Math.max(0, Math.min(1, (clientX - first.left) / (last.right - first.left)));
      setVolume(Math.ceil(ratio * VOLUME_SEGMENTS) / VOLUME_SEGMENTS);
    }

    dragTo(e.clientX);

    function onMouseMove(e: MouseEvent) { dragTo(e.clientX); }
    function onMouseUp() {
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('mouseup', onMouseUp);
    }

    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('mouseup', onMouseUp);
  }

  function saveVolume() {
    localStorage.setItem('vyu-volume', String(volume));
  }

  function loadVolume() {
    const saved = localStorage.getItem('vyu-volume');
    if (saved !== null) {
      volume = parseFloat(saved);
      if (videoEl) videoEl.volume = volume;
    }
  }

  // file functions
  async function displayFile(path: string) {
    filePath = path;
    fileName = path.split('\\').pop() || path.split('/').pop() || path;
    const ext = path.split('.').pop()?.toLowerCase() || '';
    isVideo = videoExts.includes(ext);
    fileSrc = convertFileSrc(path);
    fileSize = '';
    fileDimensions = '';
    progress = 0;
    currentTime = '0:00';
    duration = '0:00';
    playing = false;
    fileInfoLoading = true;
    fileInfoLoading = true;

    try {
      const info = await stat(path);
      fileSize = formatFileSize(info.size);
    } catch {}
  }

  function onImageLoad(e: Event) {
    const img = e.target as HTMLImageElement;
    fileDimensions = `${img.naturalWidth} × ${img.naturalHeight}`;
    fileInfoLoading = false;
  }

  function onVideoLoad() {
    if (!videoEl) return;
    fileDimensions = `${videoEl.videoWidth} × ${videoEl.videoHeight}`;
    videoEl.volume = volume;
    fileInfoLoading = false;
  }

  async function loadFile(path: string) {
    await displayFile(path);
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

  function closeFile() {
    filePath = '';
    fileSrc = '';
    fileName = 'no file open';
    isVideo = false;
    fileList = [];
    currentIndex = 0;
    playing = false;
    progress = 0;
    currentTime = '0:00';
    duration = '0:00';
    fileSize = '';
    fileDimensions = '';
  }

  // window controls
  async function minimizeWindow() {
    await getCurrentWindow().minimize();
  }

  async function maximizeWindow() {
    await getCurrentWindow().toggleMaximize();
  }

  async function closeWindow() {
    await getCurrentWindow().close();
  }

  function toggleFullscreen() {
    isFullscreen = !isFullscreen;
    if (isFullscreen) resetFsTimer();
  }

  function resetFsTimer() {
    fsControlsVisible = true;
    if (fsHideTimer) clearTimeout(fsHideTimer);
    fsHideTimer = setTimeout(() => {
      fsControlsVisible = false;
    }, 1500);
  }

  function resetZoom() {
    zoomLevel = 100;
  }

  async function startDrag(e: MouseEvent) {
    if ((e.target as HTMLElement).closest('button')) return;
    await getCurrentWindow().startDragging();
  }

  // keyboard handler
  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'f' || e.key === 'F') toggleFullscreen();
    if (e.key === 'Escape' && isFullscreen) toggleFullscreen();
    if (e.key === 'ArrowUp') { e.preventDefault(); setVolume(volume + 0.125); }
    if (e.key === 'ArrowDown') { e.preventDefault(); setVolume(volume - 0.125); }

    if (['ArrowRight', 'ArrowLeft', ' '].includes(e.key)) {
      e.preventDefault();
    }

    const videoMode = isVideo && videoEl && (hoverZone === 'video' || isFullscreen);

    if (videoMode) {
      if (e.key === ' ') togglePlay();
      if (e.key === 'ArrowRight') videoEl!.currentTime = Math.min(videoEl!.currentTime + 5, videoEl!.duration);
      if (e.key === 'ArrowLeft') videoEl!.currentTime = Math.max(videoEl!.currentTime - 5, 0);
    } else {
      if (e.key === ' ' && isVideo && videoEl) togglePlay();
      if (e.key === 'ArrowRight') navigate(1);
      if (e.key === 'ArrowLeft') navigate(-1);
    }
  }

  // file dialog
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

  // mount
  onMount(() => {
    const initial = (window as any).__INITIAL_FILE__;
    if (initial) loadFile(initial);
    loadVolume();

    const appWindow = getCurrentWindow();
    appWindow.onDragDropEvent((event) => {
      if (event.payload.type === 'drop') {
        const paths = event.payload.paths;
        if (paths?.length > 0) loadFile(paths[0]);
      }
    });

    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  });
</script>

<main
  class:fullscreen={isFullscreen}
  onmousemove={isFullscreen ? resetFsTimer : undefined}
  ondrop={(e) => e.preventDefault()}
  ondragover={(e) => e.preventDefault()}>

  <!-- topbar -->
  <div class="topbar" onmousedown={startDrag} role="toolbar" tabindex="-1">
    <span class="app-name">vyu</span>
    <span class="divider">/</span>
    <span class="filename">{fileName}</span>
    {#if fileSrc}
      <span class="divider">/</span>
      <button class="folder-btn" onclick={closeFile} aria-label="close file">⏏️</button>
      <span class="divider">/</span>
      <button class="folder-btn" onclick={openFileDialog} aria-label="open file">📁</button>
    {/if}
    <!-- window controls -->
    <div class="window-controls">
      <button class="wc-btn minimize" onclick={minimizeWindow} aria-label="minimize">−</button>
      <button class="wc-btn maximize" onclick={maximizeWindow} aria-label="maximize">▢</button>
      <button class="wc-btn close" onclick={closeWindow} aria-label="close">✕</button>
    </div>
  </div>

  <!-- main content -->
  <div class="content">

    <!-- left nav -->
    <div class="sidebar left"
      onmouseenter={() => hoverZone = 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      <button class="nav-btn" onclick={() => navigate(-1)} aria-label="previous file">‹</button>
    </div>

    <!-- viewer -->
    <div class="viewer"
      onmouseenter={() => hoverZone = isVideo ? 'video' : 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      {#if fileSrc && !isVideo}
        <img src={fileSrc} alt={fileName} onload={onImageLoad} />
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
            onloadedmetadata={onVideoLoad}
            onclick={togglePlay}>
            <track kind="captions" />
          </video>
          <!-- video controls overlay -->
          <div class="video-controls">
            <!-- progress bar -->
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
              <!-- play/pause -->
              <button class="ctrl-btn" onclick={togglePlay} aria-label="play/pause">
                {#if playing}
                  <svg width="20" height="20" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <rect x="3" y="2" width="3.5" height="12" rx="1" fill="currentColor"/>
                    <rect x="9.5" y="2" width="3.5" height="12" rx="1" fill="currentColor"/>
                  </svg>
                {:else}
                  <svg width="20" height="20" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M3 2L14 8L3 14V2Z" fill="currentColor"/>
                  </svg>
                {/if}
              </button>
              <!-- volume -->
              <div class="volume-control"
                onmouseenter={() => volumeHovered = true}
                onmouseleave={() => volumeHovered = false}
                onwheel={handleVolumeScroll}
                role="presentation">
                <button class="ctrl-btn volume-btn" onclick={toggleMute} aria-label="toggle mute">
                  {#if muted || volume === 0}
                    <svg width="18" height="18" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                      <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                      <line x1="12" y1="6" x2="16" y2="12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                      <line x1="16" y1="6" x2="12" y2="12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                    </svg>
                  {:else if volume < 0.5}
                    <svg width="18" height="18" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                      <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                      <path d="M11.5 7C12.5 7.8 13 8.4 13 9C13 9.6 12.5 10.2 11.5 11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                    </svg>
                  {:else}
                    <svg width="18" height="18" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                      <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                      <path d="M11.5 7C12.5 7.8 13 8.4 13 9C13 9.6 12.5 10.2 11.5 11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                      <path d="M13.5 5C15.5 6.5 16.5 7.7 16.5 9C16.5 10.3 15.5 11.5 13.5 13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                    </svg>
                  {/if}
                </button>
                <!-- volume diamonds -->
                {#if volumeHovered}
                  <div class="volume-diamonds" onmousedown={startVolumeDrag} role="presentation">
                    {#each Array(VOLUME_SEGMENTS) as _, i}
                      <button
                        class="volume-diamond"
                        class:filled={i < Math.round(volume * VOLUME_SEGMENTS)}
                        style="--i: {i}"
                        onclick={() => setVolume((i + 1) / VOLUME_SEGMENTS)}
                        aria-label="set volume {Math.round((i + 1) / VOLUME_SEGMENTS * 100)}%">
                      </button>
                    {/each}
                    <span class="volume-tooltip">{muted ? '0' : Math.round(volume * 100)}%</span>
                  </div>
                {/if}
              </div>
              <!-- time -->
              <span class="time-display">{currentTime} / {duration}</span>
            </div>
          </div>
        </div>
      {:else}
        <!-- empty state -->
        <button class="empty" onclick={openFileDialog}>
          <span class="empty-icon">+</span>
          <span class="empty-text">open a file</span>
        </button>
      {/if}
    </div>

    <!-- right nav -->
    <div class="sidebar right"
      onmouseenter={() => hoverZone = 'sidebar'}
      onmouseleave={() => hoverZone = 'none'}
      role="presentation">
      <button class="nav-btn" onclick={() => navigate(1)} aria-label="next file">›</button>
    </div>
  </div>

  <!-- bottombar -->
  <div class="bottombar">
    <span class="file-count">{fileList.length > 0 ? `${currentIndex + 1} / ${fileList.length}` : '—'}</span>
    <span class="file-info">
      {#if fileDimensions && fileSize}
        {fileDimensions} · {fileSize}
      {:else if !fileInfoLoading && fileName !== 'no file open'}
        {fileName}
      {:else if !fileSrc}
        no file open
      {/if}
    </span>
    <!-- zoom + fullscreen -->
    <div class="bottombar-right">
      <span class="zoom" onclick={resetZoom} role="button" tabindex="0">{zoomLevel}%</span>
      <button class="fs-btn" onclick={toggleFullscreen} aria-label="toggle fullscreen">
        <svg width="12" height="12" viewBox="0 0 12 12" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M1 4V1H4M8 1H11V4M11 8V11H8M4 11H1V8" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
        </svg>
      </button>
    </div>
  </div>

  <!-- fullscreen overlay -->
  {#if isFullscreen}
    <div class="fs-overlay" class:visible={fsControlsVisible}
      role="button"
      tabindex="0"
      onclick={(e) => { if ((e.target as HTMLElement) === e.currentTarget) togglePlay(); }}
      onkeydown={(e) => { if (e.key === ' ' || e.key === 'Enter') togglePlay(); }}>

      <!-- fs topbar -->
      <div class="fs-topbar">
        <span class="fs-filename">{fileName}</span>
        <div class="fs-window-controls">
          <button class="fs-wc-btn" onclick={minimizeWindow} aria-label="minimize">−</button>
          <button class="fs-wc-btn" onclick={maximizeWindow} aria-label="maximize">▢</button>
          <button class="fs-wc-btn close" onclick={closeWindow} aria-label="close">✕</button>
        </div>
      </div>

      <!-- fs nav buttons -->
      <div class="fs-nav-left">
        <button class="fs-nav-btn" onclick={() => navigate(-1)} aria-label="previous file">‹</button>
      </div>
      <div class="fs-nav-right">
        <button class="fs-nav-btn" onclick={() => navigate(1)} aria-label="next file">›</button>
      </div>

      <!-- fs video controls -->
      {#if isVideo && videoEl}
        <div class="fs-controls">
          <!-- fs progress bar -->
          <div class="fs-progress"
            onmousedown={startScrubbing}
            role="slider"
            aria-label="video scrubber"
            aria-valuenow={progress}
            aria-valuemin={0}
            aria-valuemax={100}
            tabindex="0">
            <div class="fs-progress-fill" style="width: {progress}%"></div>
            <div class="fs-progress-diamond" style="left: {progress}%"></div>
          </div>
          <div class="fs-controls-row">
            <!-- fs play/pause -->
            <button class="fs-ctrl-btn" onclick={togglePlay} aria-label="play/pause">
              {#if playing}
                <svg width="20" height="20" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <rect x="3" y="2" width="3.5" height="12" rx="1" fill="currentColor"/>
                  <rect x="9.5" y="2" width="3.5" height="12" rx="1" fill="currentColor"/>
                </svg>
              {:else}
                <svg width="20" height="20" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M3 2L14 8L3 14V2Z" fill="currentColor"/>
                </svg>
              {/if}
            </button>
            <!-- fs volume -->
            <div class="volume-control"
              onmouseenter={() => volumeHovered = true}
              onmouseleave={() => volumeHovered = false}
              onwheel={handleVolumeScroll}
              role="presentation">
              <button class="fs-ctrl-btn volume-btn" onclick={toggleMute} aria-label="mute">
                {#if muted || volume === 0}
                  <svg width="24" height="24" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                    <line x1="12" y1="6" x2="16" y2="12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                    <line x1="16" y1="6" x2="12" y2="12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                  </svg>
                {:else if volume < 0.5}
                  <svg width="24" height="24" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                    <path d="M11.5 7C12.5 7.8 13 8.4 13 9C13 9.6 12.5 10.2 11.5 11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                  </svg>
                {:else}
                  <svg width="24" height="24" viewBox="0 0 18 18" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M9 4L5 7H2V11H5L9 14V4Z" fill="currentColor"/>
                    <path d="M11.5 7C12.5 7.8 13 8.4 13 9C13 9.6 12.5 10.2 11.5 11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                    <path d="M13.5 5C15.5 6.5 16.5 7.7 16.5 9C16.5 10.3 15.5 11.5 13.5 13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                  </svg>
                {/if}
              </button>
              <!-- fs volume diamonds -->
              {#if volumeHovered}
                <div class="volume-diamonds" onmousedown={startVolumeDrag} role="presentation">
                  {#each Array(VOLUME_SEGMENTS) as _, i}
                    <button
                      class="volume-diamond"
                      class:filled={i < Math.round(volume * VOLUME_SEGMENTS)}
                      style="--i: {i}"
                      onclick={() => setVolume((i + 1) / VOLUME_SEGMENTS)}
                      aria-label="set volume {Math.round((i + 1) / VOLUME_SEGMENTS * 100)}%">
                    </button>
                  {/each}
                  <span class="volume-tooltip">{muted ? '0' : Math.round(volume * 100)}%</span>
                </div>
              {/if}
            </div>
            <!-- fs time -->
            <span class="fs-time">{currentTime} / {duration}</span>
            <!-- fs exit fullscreen -->
            <div class="fs-right">
              <button class="fs-ctrl-btn" onclick={toggleFullscreen} aria-label="exit fullscreen">
                <svg width="12" height="12" viewBox="0 0 12 12" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M4 1H1V4M8 1H11V4M11 8V11H8M4 11H1V8" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
                </svg>
              </button>
            </div>
          </div>
        </div>

      <!-- fs image controls -->
      {:else}
        <div class="fs-controls image-only">
          <div class="fs-controls-row">
            <span class="fs-time">{fileList.length > 0 ? `${currentIndex + 1} / ${fileList.length}` : ''}</span>
            <div class="fs-right">
              <button class="fs-ctrl-btn" onclick={toggleFullscreen} aria-label="exit fullscreen">
                <svg width="12" height="12" viewBox="0 0 12 12" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M4 1H1V4M8 1H11V4M11 8V11H8M4 11H1V8" stroke="currentColor" stroke-width="1.2" stroke-linecap="round"/>
                </svg>
              </button>
            </div>
          </div>
        </div>
      {/if}

    </div>
  {/if}

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

  /* topbar */
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
    font-size: 16px;
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

  /* window controls */
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

  /* content */
  .content {
    flex: 1;
    display: flex;
    flex-direction: row;
    overflow: hidden;
    background: #0a0a0a;
  }

  /* sidebars */
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

  /* viewer */
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

  /* empty state */
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

  /* bottombar */
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

  .file-count, .file-info {
    font-size: 12px;
    color: #444444;
    font-family: Inter, sans-serif;
  }

  .bottombar-right {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  /* zoom */
  .zoom {
    font-size: 12px;
    color: #444444;
    font-family: Inter, sans-serif;
    cursor: pointer;
    padding: 2px 4px;
    border-radius: 3px;
    transition: color 0.2s;
  }

  .zoom:hover {
    color: #888888;
  }

  /* fullscreen button */
  .fs-btn {
    background: none;
    border: none;
    color: #444444;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 4px;
    border-radius: 3px;
    transition: color 0.2s;
  }

  .fs-btn:hover {
    color: #888888;
  }

  /* video wrapper */
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
    cursor: pointer;
  }

  /* video controls overlay */
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

  /* progress bar */
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

  /* progress diamond */
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

  /* controls row */
  .controls-row {
    display: flex;
    align-items: center;
    gap: 2px;
  }

  .ctrl-btn {
    background: none;
    border: none;
    color: #cccccc;
    cursor: pointer;
    padding: 2px 4px;
    font-family: Inter, sans-serif;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
  }

  .ctrl-btn:hover {
    color: #ffffff;
  }

  .ctrl-btn svg {
    width: 28px;
    height: 28px;
  }

  /* time display */
  .time-display {
    font-size: 16px;
    color: #cccccc;
    font-family: Inter, sans-serif;
    margin-left: auto;
  }

  /* volume */
  .volume-control {
    position: relative;
    display: flex;
    align-items: center;
    gap: 4px;
    height: 28px;
  }

  .volume-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 32px;
    width: 32px;
  }

  .volume-btn svg {
    width: 28px;
    height: 28px;
  }

  /* volume diamonds */
  .volume-diamonds {
    display: flex;
    align-items: center;
    gap: 3px;
    animation: volumeBarIn 0.2s ease forwards;
    cursor: pointer;
  }

  @keyframes volumeBarIn {
    from { opacity: 0; transform: translateX(-4px); }
    to { opacity: 1; transform: translateX(0); }
  }

  .volume-diamond {
    width: 12px;
    height: 12px;
    background: none;
    border: 1px solid #555555;
    transform: rotate(45deg);
    cursor: pointer;
    padding: 0;
    transition: background 0.2s ease, border-color 0.2s ease, transform 0.2s ease;
    flex-shrink: 0;
    animation: diamondSpin 0.3s ease forwards;
    animation-delay: calc(var(--i) * 0.04s);
    opacity: 0;
  }

  @keyframes diamondSpin {
    from { opacity: 0; transform: rotate(0deg) scale(0.3); }
    to { opacity: 1; transform: rotate(45deg) scale(1); }
  }

  .volume-diamond.filled {
    background: #ffffff;
    border-color: #ffffff;
  }

  .volume-diamond:hover {
    border-color: #aaaaaa;
    transform: rotate(45deg) scale(1.2);
  }

  .volume-tooltip {
    font-size: 12px;
    color: #666666;
    font-family: Inter, sans-serif;
    margin-left: 4px;
    min-width: 28px;
  }

  /* fullscreen */
  main.fullscreen .topbar { display: none; }
  main.fullscreen .bottombar { display: none; }
  main.fullscreen .sidebar { display: none; }
  main.fullscreen .viewer { padding: 0; }
  main.fullscreen .video-controls { display: none; }

  main.fullscreen .video-wrapper {
    width: 100%;
    height: 100%;
  }

  main.fullscreen .video-wrapper video {
    max-width: 100vw;
    max-height: 100vh;
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  /* fullscreen overlay */
  .fs-overlay {
    position: fixed;
    inset: 0;
    pointer-events: none;
    opacity: 0;
    transition: opacity 0.3s;
    z-index: 100;
  }

  .fs-overlay.visible {
    opacity: 1;
    pointer-events: all;
  }

  /* fs topbar */
  .fs-topbar {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    padding: 0 16px;
    height: 40px;
    background: linear-gradient(rgba(0,0,0,0.85), transparent);
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .fs-filename {
    font-size: 12px;
    color: #888888;
    font-family: Inter, sans-serif;
  }

  .fs-window-controls {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .fs-wc-btn {
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

  .fs-wc-btn:hover {
    background: rgba(255,255,255,0.1);
    color: #cccccc;
  }

  .fs-wc-btn.close:hover {
    background: rgba(255,0,0,0.2);
    color: #ff6666;
  }

  /* fs controls */
  .fs-controls {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 12px 16px;
    background: linear-gradient(transparent, rgba(0,0,0,0.85));
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .fs-controls.image-only {
    background: linear-gradient(transparent, rgba(0,0,0,0.6));
  }

  /* fs progress bar */
  .fs-progress {
    width: 100%;
    height: 4px;
    background: rgba(255,255,255,0.2);
    border-radius: 2px;
    cursor: pointer;
    position: relative;
    overflow: visible;
  }

  .fs-progress-fill {
    height: 100%;
    background: #ffffff;
    border-radius: 2px;
    pointer-events: none;
  }

  /* fs progress diamond */
  .fs-progress-diamond {
    position: absolute;
    top: 50%;
    width: 12px;
    height: 12px;
    background: #ffffff;
    transform: translate(-50%, -50%) rotate(45deg);
    pointer-events: none;
  }

  /* fs controls row */
  .fs-controls-row {
    display: flex;
    align-items: center;
    gap: 2px;
  }

  .fs-ctrl-btn {
    background: none;
    border: none;
    color: #cccccc;
    cursor: pointer;
    font-size: 16px;
    padding: 2px 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: color 0.2s;
  }

  .fs-ctrl-btn:hover {
    color: #ffffff;
  }

  .fs-ctrl-btn svg {
    width: 20px;
    height: 20px;
  }

  .fs-ctrl-btn.volume-btn svg {
    width: 32px;
    height: 32px;
  }

  /* fs time */
  .fs-time {
    font-size: 14px;
    color: #888888;
    font-family: Inter, sans-serif;
    margin-left: 8px;
  }

  .fs-right {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  /* fs nav buttons */
  .fs-nav-left, .fs-nav-right {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .fs-nav-left { left: 16px; }
  .fs-nav-right { right: 16px; }

  .fs-nav-btn {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: rgba(0,0,0,0.5);
    border: 0.5px solid rgba(255,255,255,0.1);
    color: #666666;
    font-size: 24px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: color 0.2s, background 0.2s;
    line-height: 1;
  }

  .fs-nav-btn:hover {
    color: #cccccc;
    background: rgba(0,0,0,0.7);
  }
</style>