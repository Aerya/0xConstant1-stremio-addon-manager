<script setup>
  const props = defineProps({
    name: {
      type: String,
      required: true
    },
    idx: {
      type: Number,
      required: true
    },
    manifestURL: {
      type: String,
      required: true
    },
    logoURL: {
      type: String,
      required: false
    },
    isDeletable: {
      type: Boolean,
      required: false,
      default: true
    },
    isConfigurable: {
      type: Boolean,
      required: false,
      default: false
    },
    isResetting: {
      type: Boolean,
      required: false,
      default: false
    },
    isResetable: {
      type: Boolean,
      required: false,
      default: true
    }
  })
  
  const emits = defineEmits(['delete-addon', 'edit-manifest', 'reset-addon'])
  
  const defaultLogo = 'https://icongr.am/feather/box.svg?size=48&color=ffffff'
  
  function copyManifestURLToClipboard() {
    navigator.clipboard.writeText(props.manifestURL).then(() => {
      console.log('Text copied to clipboard')
    }).catch((error) => {
      console.error('Error copying text to clipboard', error)
    })
  }
  
  function openAddonConfigurationPage() {
    const configureURL = props.manifestURL.replace("stremio://", "https://").replace("/manifest.json", "/configure");
    window.open(configureURL);
  }
  
  function removeAddon() {
    emits('delete-addon', props.idx)
  }
  
  function openEditManifestModal() {
    emits('edit-manifest', props.idx)
  }
  
  function resetAddon() {
    emits('reset-addon', props.idx)
  }
</script>

<template>
  <div class="addon-row theme-dark">
    <div class="drag-handle" title="Drag to reorder">
      <span class="drag-icon">⋮⋮</span>
    </div>
    <div class="details">
      <img class="logo" :src="logoURL || defaultLogo" />
      <span class="name">{{ name }}</span>
    </div>
    <div class="actions">
      <button class="btn icon" title="Open addon configuration page in new window"
        :disabled="!isConfigurable" @click="openAddonConfigurationPage">
        <span class="icon-text">↗</span>
      </button>
      <button class="btn icon" title="Copy addon manifest URL to clipboard"
        @click="copyManifestURLToClipboard">
        <span class="icon-text">⧉</span>
      </button>
      <button class="btn icon" title="Edit manifest JSON" @click="openEditManifestModal">
        <span class="icon-text">✎</span>
      </button>
      <button class="btn icon reset" title="Re-install addon" 
        :disabled="!isResetable || isResetting || !manifestURL || manifestURL === 'N/A'"
        @click="resetAddon">
        <span class="spinner" v-if="isResetting"></span>
        <span class="icon-text" v-else>⟲</span>
      </button>
      <button class="btn icon danger" title="Remove addon from list" :disabled="!isDeletable"
        @click="removeAddon">
        <span class="icon-text">×</span>
      </button>
    </div>
  </div>
</template>

<style scoped>
.addon-row.theme-dark {
  --panel: #252526;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
}
.addon-row.theme-dark {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  margin: 8px 0;
  background: #2d2d30;
  border: 1px solid var(--border);
  border-radius: 10px;
  transition: all 200ms ease;
  position: relative;
}

.addon-row.theme-dark:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25);
}

.drag-handle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 40px;
  cursor: grab;
  opacity: 0.6;
  transition: opacity 200ms ease;
  user-select: none;
  touch-action: none;
}

.drag-handle:hover {
  opacity: 1;
}

.drag-handle:active {
  cursor: grabbing;
}

.drag-icon {
  font-size: 14px;
  line-height: 1;
  color: var(--muted);
  font-family: monospace;
  letter-spacing: -2px;
}

@media (hover: none) and (pointer: coarse) {
  /* Mobile/touch devices */
  .drag-handle {
    opacity: 1;
    width: 32px;
    background: var(--panel);
    border-radius: 6px;
    border: 1px solid var(--border);
  }
  
  .drag-icon {
    color: var(--text);
    font-size: 16px;
  }
  
  .addon-row.theme-dark:hover {
    transform: none;
    box-shadow: none;
  }
}
.details { display: inline-flex; align-items: center; gap: 12px; min-width: 0; }
.logo { width: 44px; height: 44px; border-radius: 10px; background: #1e1e1e; object-fit: contain; }
.name { font-weight: 600; color: var(--text); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

.actions { display: inline-flex; gap: 8px; }
.btn { 
  display: flex; 
  align-items: center; 
  justify-content: center; 
  width: 34px; 
  height: 34px; 
  border-radius: 8px; 
  border: 1px solid var(--border); 
  background: #252526; 
  color: var(--text); 
  cursor: pointer; 
  transition: all 120ms ease;
  padding: 0;
}
.btn:hover { background: #303034; border-color: #4b4b4b; transform: translateY(-1px); }
.btn.danger { background: #402326; border-color: #5a2d34; }
.btn.danger:hover { background: #4a2a2e; border-color: #6a3d44; }
.btn.reset { background: #2a3340; border-color: #3d4b5a; }
.btn.reset:hover { background: #334050; border-color: #4d5b6a; }
.btn:disabled { opacity: 0.6; cursor: not-allowed; transform: none; }
.icon-text { 
  font-size: 16px; 
  line-height: 1; 
  user-select: none; 
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  text-align: center;
  display: block;
}

/* Spinner */
.spinner {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.25);
  border-top-color: rgba(255, 255, 255, 0.95);
  animation: spin 800ms linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 600px) {
  .addon-row.theme-dark { 
    grid-template-columns: auto 1fr; 
    gap: 8px;
  }
  .actions { 
    justify-content: flex-start; 
    grid-column: 1 / -1;
    margin-top: 8px;
    padding-left: 36px;
  }
  .details {
    grid-column: 2;
  }
}
</style>
