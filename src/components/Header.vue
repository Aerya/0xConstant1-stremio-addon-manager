<script setup>
defineProps({
  addonName: {
    type: String,
    required: true
  },
  addonLogo: {
    type: String,
    required: false
  },
  addonSummary: {
    type: String,
    required: false
  },
  addonTagline: {
    type: String,
    required: false
  }
})
</script>

<template>
  <section class="header-hero theme-dark">
    <div class="container">
      <div class="hero-card">
        <div class="brand">
          <img v-if="addonLogo" :src="addonLogo" alt="Addon logo" class="logo-img" />
          <div v-else class="logo-fallback">{{ addonName?.[0] || 'S' }}</div>
        </div>
        <div class="hero-text">
          <h1 class="title">{{ addonName }}</h1>
          <p class="subtitle">{{ addonSummary }}</p>
          <small v-if="addonTagline" class="tagline"><em>{{ addonTagline }}</em></small>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
/* VS Code dark-like tokens */
.header-hero.theme-dark {
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --shadow: rgba(0, 0, 0, 0.35);
}

.header-hero.theme-dark {
  background: transparent;
  color: var(--text);
  padding: 16px;
}

.container { max-width: 960px; margin: 0 auto; }

.hero-card {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 16px;
  align-items: center;
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 16px;
  box-shadow: 0 6px 18px var(--shadow);
}

.brand {
  display: grid;
  place-items: center;
}

.logo-img {
  width: 64px;
  height: 64px;
  border-radius: 12px;
  object-fit: contain;
  background: #1e1e1e;
  border: 1px solid var(--border);
}

.logo-fallback {
  width: 64px;
  height: 64px;
  border-radius: 12px;
  display: grid;
  place-items: center;
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  color: #fff;
  font-weight: 800;
  font-size: 20px;
}

.hero-text .title {
  margin: 0 0 4px 0;
  font-size: 22px;
  font-weight: 700;
}
.subtitle {
  margin: 0;
  color: var(--muted);
  font-size: 13.5px;
}
.tagline {
  color: var(--muted);
}

@media (min-width: 1024px) {
  .hero-card { padding: 18px 20px; }
  .hero-text .title { font-size: 24px; }
}
</style>
