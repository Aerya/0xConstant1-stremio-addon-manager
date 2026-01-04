<script setup>
</script>

<template>
  <footer class="footer theme-dark" role="contentinfo">
    <div class="container">
      <div class="footer-card">
        <div class="left">
          <p class="muted credits">
            Based on
            <a href="https://github.com/pancake3000/stremio-addon-manager" target="_blank" rel="noopener">pancake3000's addon manager</a>
            — via forks by
            <a href="https://github.com/redd-ravenn/stremio-addon-manager" target="_blank" rel="noopener">redd-raven</a>
            and
            <a href="https://github.com/Viren070/stremio-addon-manager" target="_blank" rel="noopener">Viren070</a>.
            Thanks also to Sleeyax &amp; &lt;Code/&gt;.
          </p>
        </div>
        <div class="right">
          <a class="btn-link" href="https://github.com/0xConstant1/stremio-addon-manager" target="_blank" rel="noopener">Source Code</a>
        </div>
      </div>
    </div>
  </footer>
</template>

<style scoped>
.footer.theme-dark {
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
}

.footer.theme-dark {
  background: transparent;
  color: var(--text);
  padding: 16px 12px;
  font-size: 1.3rem;
}

.container { max-width: 960px; margin: 0 auto; }

.footer-card {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  gap: 12px;
  background: linear-gradient(0deg, rgba(255,255,255,0.02), rgba(255,255,255,0.02)), var(--panel);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 12px 14px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}
.left { display: flex; flex-direction: column; gap: 6px; }
.right { display: flex; align-items: center; }

.credits { line-height: 1.5; word-break: break-word; }

.muted { color: var(--muted); margin: 0; }
.credits a { color: #8fc7ff; text-decoration: underline; text-underline-offset: 2px; }
.credits a:hover { color: #b6dbff; }

.btn-link {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--panel-2);
  color: var(--text);
  text-decoration: none;
  transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;
}
.btn-link:hover { background: #323234; border-color: #4a4a4a; color: #ffffff; }

@media (max-width: 640px) {
  .footer.theme-dark { font-size: 1.1rem; }
  .container { padding: 0 12px; }
  .footer-card {
    grid-template-columns: 1fr;
    gap: 10px;
    padding: 10px 12px;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.12);
  }
  .credits { text-align: justify; text-justify: inter-word; }
  .right { justify-content: stretch; }
  .btn-link { width: 100%; justify-content: center; padding: 10px 14px; }
}
</style>
