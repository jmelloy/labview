<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { useRoute, useRouter } from "vue-router";
import { useNotebooksStore } from "@/stores/notebooks";
import { pagesApi, entriesApi } from "@/api";
import type { Page, Entry } from "@/types";

const route = useRoute();
const router = useRouter();
const notebooksStore = useNotebooksStore();

const page = ref<Page | null>(null);
const entries = ref<Entry[]>([]);
const loading = ref(true);
const executingEntries = ref<Set<string>>(new Set());

// Variation modal state
const showVariationModal = ref(false);
const variationEntry = ref<Entry | null>(null);
const variationTitle = ref("");
const creatingVariation = ref(false);

// Lineage modal state
const showLineageModal = ref(false);
const lineageEntry = ref<Entry | null>(null);
const lineageData = ref<{ ancestors: Entry[]; descendants: Entry[]; entry: Entry } | null>(null);
const loadingLineage = ref(false);

const notebookId = computed(() => route.params.notebookId as string);
const pageId = computed(() => route.params.pageId as string);
const notebook = computed(() => notebooksStore.notebooks.get(notebookId.value));

onMounted(async () => {
  await notebooksStore.loadNotebook(notebookId.value);
  try {
    page.value = await pagesApi.get(notebooksStore.workspacePath, pageId.value);
    entries.value = await entriesApi.list(
      notebooksStore.workspacePath,
      pageId.value,
    );
  } catch (e) {
    console.error("Failed to load page:", e);
  } finally {
    loading.value = false;
  }
});

const getStatusClass = (status: string) => {
  return (
    {
      created: "status-created",
      running: "status-running",
      completed: "status-completed",
      failed: "status-failed",
    }[status] || ""
  );
};

const getEntryTypeIcon = (entryType: string) => {
  const icons: Record<string, string> = {
    custom: "📝",
    api_call: "🌐",
    database_query: "🗃️",
    graphql: "◈",
  };
  return icons[entryType] || "📄";
};

const createNewEntry = () => {
  router.push(
    `/notebooks/${notebookId.value}/pages/${pageId.value}/entries/new`,
  );
};

async function executeEntry(entry: Entry) {
  executingEntries.value.add(entry.id);
  try {
    const updated = await entriesApi.execute(notebooksStore.workspacePath, entry.id);
    const idx = entries.value.findIndex(e => e.id === entry.id);
    if (idx >= 0) {
      entries.value[idx] = updated;
    }
  } catch (e) {
    console.error("Failed to execute entry:", e);
    alert("Failed to execute entry: " + (e instanceof Error ? e.message : "Unknown error"));
  } finally {
    executingEntries.value.delete(entry.id);
  }
}

function openVariationModal(entry: Entry) {
  variationEntry.value = entry;
  variationTitle.value = `${entry.title} - Variation`;
  showVariationModal.value = true;
}

async function createVariation() {
  if (!variationEntry.value || !variationTitle.value.trim()) return;

  creatingVariation.value = true;
  try {
    const response = await fetch(`/api/entries/${variationEntry.value.id}/variations`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        workspace_path: notebooksStore.workspacePath,
        title: variationTitle.value,
        input_overrides: {},
        tags: [],
      }),
    });

    if (!response.ok) throw new Error("Failed to create variation");

    const newEntry = await response.json();
    entries.value.push(newEntry);
    showVariationModal.value = false;
    variationEntry.value = null;
    variationTitle.value = "";
  } catch (e) {
    console.error("Failed to create variation:", e);
    alert("Failed to create variation");
  } finally {
    creatingVariation.value = false;
  }
}

async function openLineageModal(entry: Entry) {
  lineageEntry.value = entry;
  showLineageModal.value = true;
  loadingLineage.value = true;

  try {
    lineageData.value = await entriesApi.getLineage(notebooksStore.workspacePath, entry.id);
  } catch (e) {
    console.error("Failed to load lineage:", e);
    lineageData.value = null;
  } finally {
    loadingLineage.value = false;
  }
}

function closeLineageModal() {
  showLineageModal.value = false;
  lineageEntry.value = null;
  lineageData.value = null;
}

function closeVariationModal() {
  showVariationModal.value = false;
  variationEntry.value = null;
  variationTitle.value = "";
}
</script>

<template>
  <div class="page-detail" v-if="page">
    <header class="page-header">
      <div class="breadcrumb">
        <RouterLink to="/notebooks">Notebooks</RouterLink>
        <span>/</span>
        <RouterLink :to="`/notebooks/${notebookId}`">{{
          notebook?.title
        }}</RouterLink>
        <span>/</span>
        <span>{{ page.title }}</span>
      </div>

      <div class="title-row">
        <h1>{{ page.title }}</h1>
        <span class="date">{{
          page.date ? new Date(page.date).toLocaleDateString() : "Undated"
        }}</span>
      </div>

      <div v-if="page.tags.length > 0" class="tags">
        <span v-for="tag in page.tags" :key="tag" class="tag">{{ tag }}</span>
      </div>
    </header>

    <section
      class="narrative-section card"
      v-if="Object.values(page.narrative || {}).some(Boolean)"
    >
      <h2>Narrative</h2>
      <div class="narrative-fields">
        <div v-if="page.narrative?.goals" class="narrative-field">
          <h4>Goals</h4>
          <p>{{ page.narrative.goals }}</p>
        </div>
        <div v-if="page.narrative?.hypothesis" class="narrative-field">
          <h4>Hypothesis</h4>
          <p>{{ page.narrative.hypothesis }}</p>
        </div>
        <div v-if="page.narrative?.observations" class="narrative-field">
          <h4>Observations</h4>
          <p>{{ page.narrative.observations }}</p>
        </div>
        <div v-if="page.narrative?.conclusions" class="narrative-field">
          <h4>Conclusions</h4>
          <p>{{ page.narrative.conclusions }}</p>
        </div>
        <div v-if="page.narrative?.next_steps" class="narrative-field">
          <h4>Next Steps</h4>
          <p>{{ page.narrative.next_steps }}</p>
        </div>
      </div>
    </section>

    <section class="entries-section">
      <div class="section-header">
        <h2>Entries ({{ entries.length }})</h2>
        <button class="btn btn-primary" @click="createNewEntry">
          + New Entry
        </button>
      </div>

      <div v-if="entries.length === 0" class="empty">
        <p>No entries in this page yet.</p>
        <p>Create your first entry to start documenting your work.</p>
        <button class="btn btn-primary" @click="createNewEntry">
          Create Entry
        </button>
      </div>

      <div v-else class="entries-list">
        <div v-for="entry in entries" :key="entry.id" class="entry-card card">
          <div class="entry-header">
            <div class="entry-info">
              <span class="entry-type">
                <span class="entry-type-icon">{{
                  getEntryTypeIcon(entry.entry_type)
                }}</span>
                {{ entry.entry_type }}
              </span>
              <h3>{{ entry.title }}</h3>
            </div>
            <span class="entry-status" :class="getStatusClass(entry.status)">
              {{ entry.status }}
            </span>
          </div>

          <div class="entry-meta">
            <span
              >Created: {{ new Date(entry.created_at).toLocaleString() }}</span
            >
            <span v-if="entry.parent_id">Parent: {{ entry.parent_id }}</span>
          </div>

          <div v-if="entry.metadata?.tags?.length" class="tags">
            <span v-for="tag in entry.metadata.tags" :key="tag" class="tag">{{
              tag
            }}</span>
          </div>

          <div class="entry-actions">
            <button 
              class="btn btn-secondary" 
              v-if="entry.status === 'created'"
              @click="executeEntry(entry)"
              :disabled="executingEntries.has(entry.id)"
            >
              {{ executingEntries.has(entry.id) ? "Executing..." : "Execute" }}
            </button>
            <button class="btn btn-secondary" @click="openVariationModal(entry)">
              Create Variation
            </button>
            <button class="btn btn-secondary" @click="openLineageModal(entry)">
              View Lineage
            </button>
          </div>
        </div>
      </div>
    </section>

    <!-- Variation Modal -->
    <div v-if="showVariationModal" class="modal-overlay" @click.self="closeVariationModal">
      <div class="modal">
        <h2>Create Variation</h2>
        <p class="modal-subtitle">Based on: {{ variationEntry?.title }}</p>
        <form @submit.prevent="createVariation">
          <div class="form-group">
            <label for="variation-title">Title *</label>
            <input
              id="variation-title"
              v-model="variationTitle"
              type="text"
              required
            />
          </div>
          <div class="modal-actions">
            <button type="button" class="btn" @click="closeVariationModal">Cancel</button>
            <button type="submit" class="btn btn-primary" :disabled="creatingVariation">
              {{ creatingVariation ? "Creating..." : "Create Variation" }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- Lineage Modal -->
    <div v-if="showLineageModal" class="modal-overlay" @click.self="closeLineageModal">
      <div class="modal modal-wide">
        <h2>Entry Lineage</h2>
        <p class="modal-subtitle">{{ lineageEntry?.title }}</p>
        
        <div v-if="loadingLineage" class="loading">Loading lineage...</div>
        
        <div v-else-if="lineageData" class="lineage-content">
          <div class="lineage-section" v-if="lineageData.ancestors.length > 0">
            <h4>Ancestors</h4>
            <div class="lineage-list">
              <div v-for="ancestor in lineageData.ancestors" :key="ancestor.id" class="lineage-item">
                {{ ancestor.title }} <span class="lineage-status">{{ ancestor.status }}</span>
              </div>
            </div>
          </div>
          
          <div class="lineage-section lineage-current">
            <h4>Current Entry</h4>
            <div class="lineage-item current">
              {{ lineageData.entry.title }} <span class="lineage-status">{{ lineageData.entry.status }}</span>
            </div>
          </div>
          
          <div class="lineage-section" v-if="lineageData.descendants.length > 0">
            <h4>Descendants</h4>
            <div class="lineage-list">
              <div v-for="desc in lineageData.descendants" :key="desc.id" class="lineage-item">
                {{ desc.title }} <span class="lineage-status">{{ desc.status }}</span>
              </div>
            </div>
          </div>
          
          <div v-if="lineageData.ancestors.length === 0 && lineageData.descendants.length === 0" class="empty">
            No lineage found. This entry has no parent or children.
          </div>
        </div>
        
        <div class="modal-actions">
          <button type="button" class="btn" @click="closeLineageModal">Close</button>
        </div>
      </div>
    </div>
  </div>

  <div v-else-if="loading" class="loading">Loading page...</div>
</template>

<style scoped>
.breadcrumb {
  display: flex;
  gap: 0.5rem;
  color: var(--color-text-secondary);
  margin-bottom: 1rem;
}

.breadcrumb a {
  color: var(--color-text-secondary);
}

.breadcrumb a:hover {
  color: var(--color-primary);
}

.title-row {
  display: flex;
  align-items: baseline;
  gap: 1rem;
  margin-bottom: 0.5rem;
}

.date {
  color: var(--color-text-secondary);
  font-size: 1rem;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.page-header {
  margin-bottom: 2rem;
}

.narrative-section {
  margin-bottom: 2rem;
}

.narrative-section h2 {
  margin-bottom: 1rem;
}

.narrative-fields {
  display: grid;
  gap: 1rem;
}

.narrative-field h4 {
  color: var(--color-text-secondary);
  font-size: 0.875rem;
  margin-bottom: 0.25rem;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.entries-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.entry-card {
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}

.entry-card:hover {
  box-shadow: var(--shadow-md);
}

.entry-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 0.75rem;
}

.entry-type {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.75rem;
  text-transform: uppercase;
  color: var(--color-text-secondary);
  letter-spacing: 0.05em;
}

.entry-info h3 {
  margin-top: 0.25rem;
}

.entry-status {
  padding: 0.25rem 0.75rem;
  border-radius: var(--radius-sm);
  font-size: 0.875rem;
  font-weight: 500;
}

.status-created {
  background: #e0e7ff;
  color: #4338ca;
}

.status-running {
  background: #fef3c7;
  color: #b45309;
}

.status-completed {
  background: #dcfce7;
  color: #15803d;
}

.status-failed {
  background: #fee2e2;
  color: #b91c1c;
}

.entry-meta {
  display: flex;
  gap: 1.5rem;
  font-size: 0.875rem;
  color: var(--color-text-secondary);
  margin-bottom: 0.75rem;
}

.entry-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid var(--color-border);
}

.loading,
.empty {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-secondary);
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: var(--color-background);
  padding: 2rem;
  border-radius: 8px;
  width: 100%;
  max-width: 500px;
  box-shadow: var(--shadow-lg);
}

.modal-wide {
  max-width: 700px;
}

.modal h2 {
  margin-bottom: 0.5rem;
}

.modal-subtitle {
  color: var(--color-text-secondary);
  margin-bottom: 1.5rem;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--color-border);
  border-radius: 4px;
  font-size: 1rem;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 1.5rem;
}

.lineage-content {
  margin: 1rem 0;
}

.lineage-section {
  margin-bottom: 1.5rem;
}

.lineage-section h4 {
  color: var(--color-text-secondary);
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

.lineage-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.lineage-item {
  padding: 0.75rem;
  background: var(--color-surface);
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.lineage-item.current {
  border: 2px solid var(--color-primary);
}

.lineage-status {
  font-size: 0.75rem;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  background: var(--color-border);
}
</style>
