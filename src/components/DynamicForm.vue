<template>
  <form class="dyn-form theme-dark" @submit.prevent="handleSubmit">
    <div v-if="!isAdvancedMode">
      <div class="form-row">
        <label for="name">Name</label>
        <input id="name" name="name" type="text" v-model="formModel.name" class="input" />
      </div>

      <div class="form-row">
        <label for="description">Description</label>
        <textarea id="description" name="description" rows="4" v-model="formModel.description" class="input textarea"></textarea>
      </div>

      <div class="form-row">
        <label for="logo">Logo URL</label>
        <input id="logo" name="logo" type="text" v-model="formModel.logo" class="input" />
      </div>

      <div class="form-row">
        <label for="background">Background URL</label>
        <input id="background" name="background" type="text" v-model="formModel.background" class="input" />
      </div>

      <div v-if="formModel.catalogs && formModel.catalogs.length > 0" class="form-row">
        <label>Catalogs</label>
        <div class="catalog-list">
          <div v-for="(catalog, index) in formModel.catalogs" :key="catalog.type" class="catalog-item">
            <label :for="'catalog-' + catalog.type" class="catalog-type-label">{{ catalog.type }}</label>
            <input :id="'catalog-' + catalog.type" type="text" v-model="catalog.name" placeholder="Catalog Name" class="input" />
            <button type="button" class="btn danger" @click="removeCatalog(index)">
              <img src="https://icongr.am/feather/trash-2.svg?size=16" alt="Delete Catalog" />
            </button>
          </div>
        </div>
      </div>

      <div class="form-actions">
        <button class="btn primary" type="submit">Save</button>
        <button type="button" class="btn accent" @click="toggleEditMode">Advanced mode</button>
      </div>
    </div>

    <div v-else>
      <textarea v-model="jsonModel" rows="10" class="input textarea json-editor"></textarea>
      <div class="form-actions">
        <button class="btn primary" type="button" @click="updateFromJson">Save</button>
        <button type="button" class="btn" @click="toggleEditMode">Classic mode</button>
      </div>
    </div>
  </form>
</template>

<script setup>
import { ref, watch, defineProps, defineEmits, onMounted } from 'vue'

const props = defineProps({
  manifest: {
    type: Object,
    required: true
  }
})

const emits = defineEmits(['update-manifest'])

const isAdvancedMode = ref(false);
const formModel = ref({
  name: '',
  description: '',
  logo: '',
  background: '',
  catalogs: []
});
const jsonModel = ref('')

watch(() => props.manifest, (newManifest) => {
  formModel.value = JSON.parse(JSON.stringify(newManifest));
  jsonModel.value = JSON.stringify(newManifest, null, 2);
}, { immediate: true });

onMounted(() => {
  calculateMaxLabelWidth();
});

function calculateMaxLabelWidth() {
  const labels = document.querySelectorAll('.catalog-type-label');
  let maxWidth = 0;

  labels.forEach(label => {
    maxWidth = Math.max(maxWidth, label.scrollWidth);
  });

  labels.forEach(label => {
    label.style.width = `${maxWidth}px`;
  });
}

function toggleEditMode() {
  isAdvancedMode.value = !isAdvancedMode.value;
  if (!isAdvancedMode.value) {
    try {
      formModel.value = JSON.parse(jsonModel.value);
      calculateMaxLabelWidth();
    } catch (e) {
      alert('Invalid JSON format');
    }
  }
}

function handleSubmit() {
  emits('update-manifest', formModel.value);
}

function removeCatalog(index) {
  if (Array.isArray(formModel.value.catalogs)) {
    formModel.value.catalogs.splice(index, 1);
  }
}

function updateFromJson() {
  try {
    formModel.value = JSON.parse(jsonModel.value);
    emits('update-manifest', formModel.value);
    isAdvancedMode.value = false;
  } catch (e) {
    alert('Invalid JSON format');
  }
}
</script>

<style scoped>
.dyn-form.theme-dark {
  --panel: #252526;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  background: var(--panel);
  padding: 4px; /* Ensure there's some padding to see the background */
  border-radius: 8px;
}
.dyn-form { 
  display: flex; 
  flex-direction: column; 
  gap: 14px; 
  min-height: 100%;
  padding-bottom: 8px;
}
.form-row { display: grid; gap: 8px; }
label { font-weight: 600; font-size: 13px; color: var(--text); }
.input { width: 100%; padding: 10px 12px; border-radius: 8px; border: 1px solid var(--border); background: #1e1e1e; color: var(--text); }
.textarea { resize: vertical; min-height: 90px; }
.json-editor { 
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace; 
  font-size: 13px; 
  line-height: 1.4;
  background: #1e1e1e !important;
  color: var(--text) !important;
}

.catalog-list { display: flex; flex-direction: column; gap: 8px; }
.catalog-item { display: grid; grid-template-columns: auto 1fr auto; gap: 8px; align-items: center; }
.catalog-type-label { color: var(--text); font-weight: 700; min-width: 100px; text-align: right; }
.btn { display: inline-grid; place-items: center; padding: 8px 10px; border-radius: 8px; border: 1px solid var(--border); background: #2d2d30; color: var(--text); cursor: pointer; }
.btn.primary { background: linear-gradient(180deg, #3a3a3a, #2f2f2f); border-color: #4a4a4a; }
.btn.accent { background: linear-gradient(180deg, #007acc, #0e639c); border-color: transparent; color: #fff; }
.btn.danger { background: #402326; border-color: #5a2d34; }
.form-actions { 
  display: flex; 
  gap: 10px; 
  margin-top: 24px;
  padding-top: 16px;
  border-top: 1px solid var(--border);
}

/* Mobile responsiveness for form */
@media (max-width: 768px) {
  .catalog-item { 
    grid-template-columns: 1fr; 
    gap: 6px;
  }
  .catalog-type-label { 
    text-align: left;
    min-width: auto;
    font-size: 12px;
  }
  
  .form-actions {
    flex-direction: column;
    gap: 12px;
    margin-top: 20px;
    padding-top: 14px;
  }
  
  .btn {
    width: 100%;
    justify-content: center;
    padding: 12px 16px;
  }
}

@media (max-width: 480px) {
  .dyn-form {
    gap: 12px;
  }
  
  .input, .textarea {
    font-size: 16px; /* Prevents zoom on iOS */
  }
  
  .form-actions {
    margin-top: 16px;
    padding-top: 12px;
  }
}
</style>
