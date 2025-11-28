<script setup lang="ts">
import { onMounted, ref } from "vue";
import { useRouter } from "vue-router";
import { useNotebooksStore } from "@/stores/notebooks";

const notebooksStore = useNotebooksStore();
const router = useRouter();

const showModal = ref(false);
const newNotebook = ref({
  title: "",
  description: "",
  tags: "",
});
const creating = ref(false);

onMounted(() => {
  notebooksStore.loadNotebooks();
});

async function handleCreateNotebook() {
  if (!newNotebook.value.title.trim()) return;

  creating.value = true;
  const tags = newNotebook.value.tags
    .split(",")
    .map((t) => t.trim())
    .filter(Boolean);

  const notebook = await notebooksStore.createNotebook(
    newNotebook.value.title,
    newNotebook.value.description,
    tags
  );

  creating.value = false;

  if (notebook) {
    showModal.value = false;
    newNotebook.value = { title: "", description: "", tags: "" };
    router.push(`/notebooks/${notebook.id}`);
  }
}

function closeModal() {
  showModal.value = false;
  newNotebook.value = { title: "", description: "", tags: "" };
}
</script>

<template>
  <div class="notebooks-view">
    <header class="page-header">
      <h1>Notebooks</h1>
      <button class="btn btn-primary" @click="showModal = true">+ New Notebook</button>
    </header>

    <div v-if="notebooksStore.loading" class="loading">
      Loading notebooks...
    </div>

    <div v-else-if="notebooksStore.error" class="error">
      {{ notebooksStore.error }}
    </div>

    <div v-else-if="notebooksStore.notebooksList.length === 0" class="empty">
      <p>No notebooks yet.</p>
      <p>Create your first notebook to get started.</p>
    </div>

    <div v-else class="notebooks-grid">
      <RouterLink
        v-for="notebook in notebooksStore.notebooksList"
        :key="notebook.id"
        :to="`/notebooks/${notebook.id}`"
        class="notebook-card card"
      >
        <h3>{{ notebook.title }}</h3>
        <p v-if="notebook.description" class="description">
          {{ notebook.description }}
        </p>
        <div v-if="notebook.tags.length > 0" class="tags">
          <span v-for="tag in notebook.tags" :key="tag" class="tag">
            {{ tag }}
          </span>
        </div>
        <div class="meta">
          Created: {{ new Date(notebook.created_at).toLocaleDateString() }}
        </div>
      </RouterLink>
    </div>

    <!-- Create Notebook Modal -->
    <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
      <div class="modal">
        <h2>Create New Notebook</h2>
        <form @submit.prevent="handleCreateNotebook">
          <div class="form-group">
            <label for="title">Title *</label>
            <input
              id="title"
              v-model="newNotebook.title"
              type="text"
              placeholder="My Notebook"
              required
            />
          </div>
          <div class="form-group">
            <label for="description">Description</label>
            <textarea
              id="description"
              v-model="newNotebook.description"
              placeholder="What is this notebook about?"
              rows="3"
            ></textarea>
          </div>
          <div class="form-group">
            <label for="tags">Tags (comma-separated)</label>
            <input
              id="tags"
              v-model="newNotebook.tags"
              type="text"
              placeholder="experiment, ml, research"
            />
          </div>
          <div class="modal-actions">
            <button type="button" class="btn" @click="closeModal">Cancel</button>
            <button type="submit" class="btn btn-primary" :disabled="creating">
              {{ creating ? "Creating..." : "Create Notebook" }}
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.notebooks-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
}

.notebook-card {
  display: block;
  text-decoration: none;
  color: inherit;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}

.notebook-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
  text-decoration: none;
}

.notebook-card h3 {
  margin-bottom: 0.5rem;
  color: var(--color-primary);
}

.description {
  color: var(--color-text-secondary);
  margin-bottom: 1rem;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.meta {
  font-size: 0.875rem;
  color: var(--color-text-secondary);
}

.loading,
.error,
.empty {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-secondary);
}

.error {
  color: var(--color-error);
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

.modal h2 {
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
</style>
