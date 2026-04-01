<script lang="ts">
</script>

<main>
  <div class="topbar">
    <span class="app-name">vyu</span>
    <span class="divider">/</span>
    <span class="filename">no file open</span>
  </div>

  <div class="viewer">
  </div>

  <div class="bottombar">
    <span class="file-count">—</span>
    <span class="file-info">open a file to get started</span>
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

  .viewer {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #0a0a0a;
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
</style>