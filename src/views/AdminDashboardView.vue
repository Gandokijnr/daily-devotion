<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import { supabase } from '@/lib/supabaseClient'

interface Devotion {
  id: number
  title: string
  verse: string
  content: string
  date: string
  createdAt: string
}

const router = useRouter()

const devotions = ref<Devotion[]>([])
const loading = ref(true)
const error = ref<string | null>(null)

const title = ref('')
const date = ref<string>(new Date().toISOString().slice(0, 10))
const verse = ref('')
const content = ref('')
const submitting = ref(false)
const editingId = ref<number | null>(null)

const editorEl = ref<HTMLElement | null>(null)

const adminEmail = ref<string | null>(null)

function formatDate(value: string) {
  const d = new Date(value)
  if (Number.isNaN(d.getTime())) return value
  return d.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' })
}

watch(content, (value) => {
  if (editorEl.value && editorEl.value.innerHTML !== (value ?? '')) {
    editorEl.value.innerHTML = value ?? ''
  }
})

function focusEditor() {
  if (!editorEl.value) return
  editorEl.value.focus()
}

function applyFormat(command: string, value?: string) {
  if (typeof document === 'undefined') return
  focusEditor()
  document.execCommand(command, false, value)
  if (editorEl.value) {
    content.value = editorEl.value.innerHTML
  }
}

function onEditorInput() {
  if (editorEl.value) {
    content.value = editorEl.value.innerHTML
  }
}

async function loadSession() {
  const { data } = await supabase.auth.getUser()
  adminEmail.value = data.user?.email ?? null
}

async function loadDevotions() {
  loading.value = true
  error.value = null

  const { data, error: queryError } = await supabase
    .from('devotions')
    .select('*')
    .order('createdAt', { ascending: false })

  if (queryError) {
    error.value = 'Unable to load devotions.'
    loading.value = false
    return
  }

  devotions.value = (data ?? []) as Devotion[]
  loading.value = false
}

function resetForm() {
  title.value = ''
  verse.value = ''
  content.value = ''
  date.value = new Date().toISOString().slice(0, 10)
  editingId.value = null
}

async function createDevotion() {
  submitting.value = true
  error.value = null

  const devotionDate = new Date(date.value)

  const { error: insertError } = await supabase
    .from('devotions')
    .insert({
      title: title.value.trim(),
      verse: verse.value.trim(),
      content: content.value.trim(),
      date: devotionDate.toISOString(),
      createdAt: new Date().toISOString(),
    })

  if (insertError) {
    submitting.value = false
    alert('Unable to publish devotion. Please try again.')
    return
  }

  alert('Devotion published successfully.')

  resetForm()
  submitting.value = false
  loadDevotions()
}

function startEdit(devotion: Devotion) {
  editingId.value = devotion.id
  title.value = devotion.title
  verse.value = devotion.verse
  content.value = devotion.content

  const d = new Date(devotion.date)
  if (!Number.isNaN(d.getTime())) {
    date.value = d.toISOString().slice(0, 10)
  }
}

async function updateDevotion() {
  if (!editingId.value) return

  submitting.value = true
  error.value = null

  const devotionDate = new Date(date.value)

  const { error: updateError } = await supabase
    .from('devotions')
    .update({
      title: title.value.trim(),
      verse: verse.value.trim(),
      content: content.value.trim(),
      date: devotionDate.toISOString(),
    })
    .eq('id', editingId.value)

  if (updateError) {
    submitting.value = false
    alert('Unable to update devotion. Please try again.')
    return
  }

  const index = devotions.value.findIndex((d) => d.id === editingId.value)
  if (index !== -1) {
    const existing = devotions.value[index]!
    devotions.value[index] = {
      ...existing,
      title: title.value.trim(),
      verse: verse.value.trim(),
      content: content.value.trim(),
      date: devotionDate.toISOString(),
    }
  }

  alert('Devotion updated successfully.')

  resetForm()
  submitting.value = false
}

function cancelEdit() {
  resetForm()
}

async function deleteDevotion(devotion: Devotion) {
  const confirmed = window.confirm('Delete this devotion permanently?')
  if (!confirmed) return

  const { error: deleteError } = await supabase
    .from('devotions')
    .delete()
    .eq('id', devotion.id)

  if (deleteError) {
    alert('Unable to delete devotion.')
    return
  }

  devotions.value = devotions.value.filter((d) => d.id !== devotion.id)
}

async function signOut() {
  await supabase.auth.signOut()
  router.push({ name: 'admin-login' })
}

onMounted(async () => {
  await loadSession()
  await loadDevotions()

  if (editorEl.value) {
    editorEl.value.innerHTML = content.value ?? ''
  }
})
</script>

<template>
  <div class="min-h-screen bg-slate-50 px-4 py-6 sm:px-6 lg:px-8">
    <div class="max-w-6xl mx-auto space-y-6">
      <header class="flex items-center justify-between">
        <div>
          <p class="text-xs font-semibold tracking-[0.2em] text-primary-600 uppercase mb-1">GraceDaily Admin</p>
          <h1 class="text-xl font-semibold text-slate-900">Devotional Manager</h1>
          <p class="text-xs text-slate-500" v-if="adminEmail">Signed in as {{ adminEmail }}</p>
        </div>
        <button
          type="button"
          class="text-xs text-slate-500 hover:text-slate-700"
          @click="signOut"
        >
          Sign out
        </button>
      </header>

      <div class="grid gap-6 lg:grid-cols-[minmax(0,_2fr)_minmax(0,_3fr)]">
        <section class="gd-card p-5 space-y-4">
          <h2 class="text-sm font-semibold text-slate-900">
            <span v-if="editingId">Edit devotion</span>
            <span v-else>New devotion</span>
          </h2>

          <div class="space-y-1">
            <label class="block text-xs font-medium text-slate-700">Title</label>
            <input
              v-model="title"
              type="text"
              required
              class="block w-full rounded-lg border border-slate-200 bg-slate-50 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
            >
          </div>

          <div class="grid grid-cols-1 sm:grid-cols-2 gap-3">
            <div class="space-y-1">
              <label class="block text-xs font-medium text-slate-700">Date</label>
              <input
                v-model="date"
                type="date"
                required
                class="block w-full rounded-lg border border-slate-200 bg-slate-50 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              >
            </div>

            <div class="space-y-1">
              <label class="block text-xs font-medium text-slate-700">Memory verse</label>
              <input
                v-model="verse"
                type="text"
                required
                placeholder="John 3:16 | For God so loved the world..."
                class="block w-full rounded-lg border border-slate-200 bg-slate-50 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
              >
            </div>
          </div>

          <div class="space-y-1">
            <label class="block text-xs font-medium text-slate-700">Main content</label>

            <div class="rounded-lg border border-slate-200 bg-white">
              <div class="flex flex-wrap items-center gap-1 border-b border-slate-200 bg-slate-50 px-2 py-1 text-[11px] text-slate-700">
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100 font-semibold"
                  @click="applyFormat('bold')"
                >
                  B
                </button>
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100 italic"
                  @click="applyFormat('italic')"
                >
                  I
                </button>
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100 underline"
                  @click="applyFormat('underline')"
                >
                  U
                </button>

                <span class="mx-1 h-4 w-px bg-slate-200" />

                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100"
                  @click="applyFormat('justifyLeft')"
                >
                  Left
                </button>
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100"
                  @click="applyFormat('justifyCenter')"
                >
                  Center
                </button>
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100"
                  @click="applyFormat('justifyRight')"
                >
                  Right
                </button>

                <span class="mx-1 h-4 w-px bg-slate-200" />

                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100"
                  @click="applyFormat('insertUnorderedList')"
                >
                  • List
                </button>
                <button
                  type="button"
                  class="px-1.5 py-0.5 rounded hover:bg-slate-100"
                  @click="applyFormat('insertOrderedList')"
                >
                  1. List
                </button>
              </div>

              <div
                ref="editorEl"
                class="admin-editor-content min-h-[180px] max-h-[420px] overflow-y-auto px-3 py-2 text-sm text-slate-900 focus:outline-none"
                contenteditable="true"
                @input="onEditorInput"
              ></div>
            </div>

            <p class="text-[11px] text-slate-400">You can paste from your prepared sermon notes or write directly here.</p>
          </div>

          <button
            type="button"
            class="gd-btn-primary w-full text-sm"
            :disabled="submitting"
            @click="editingId ? updateDevotion() : createDevotion()"
          >
            <span v-if="submitting">
              <span v-if="editingId">Updating...</span>
              <span v-else>Publishing...</span>
            </span>
            <span v-else>
              <span v-if="editingId">Update devotion</span>
              <span v-else>Publish devotion</span>
            </span>
          </button>
          <button
            v-if="editingId"
            type="button"
            class="mt-2 w-full text-xs text-slate-500 hover:text-slate-700"
            @click="cancelEdit"
          >
            Cancel editing
          </button>
        </section>

        <section class="space-y-3">
          <div class="flex items-center justify-between">
            <h2 class="text-sm font-semibold text-slate-900">Existing devotions</h2>
            <p class="text-xs text-slate-500" v-if="!loading && devotions.length">
              {{ devotions.length }} total
            </p>
          </div>

          <div v-if="loading" class="text-xs text-slate-500">Loading devotions...</div>
          <div v-else-if="error" class="text-xs text-red-600">{{ error }}</div>
          <div v-else-if="!devotions.length" class="text-xs text-slate-500">
            No devotions have been created yet.
          </div>

          <div v-else class="space-y-2 max-h-[520px] overflow-y-auto pr-1">
            <article
              v-for="devotion in devotions"
              :key="devotion.id"
              class="gd-card px-4 py-3 flex items-start justify-between gap-3"
            >
              <div class="space-y-1">
                <p class="text-[11px] text-slate-500">{{ formatDate(devotion.date) }}</p>
                <h3 class="text-sm font-medium text-slate-900 line-clamp-1">{{ devotion.title }}</h3>
                <p class="text-[11px] text-slate-500 italic line-clamp-1">{{ devotion.verse }}</p>
              </div>
              <div class="flex items-center gap-2">
                <button
                  type="button"
                  class="text-[11px] text-primary-600 hover:text-primary-700"
                  @click="startEdit(devotion)"
                >
                  Edit
                </button>
                <button
                  type="button"
                  class="text-[11px] text-red-500 hover:text-red-600"
                  @click="deleteDevotion(devotion)"
                >
                  Delete
                </button>
              </div>
            </article>
          </div>
        </section>
      </div>
    </div>
  </div>
</template>

<style scoped>
.admin-editor-content :deep(ul) {
  list-style-type: disc;
  padding-left: 1.25rem;
}

.admin-editor-content :deep(ol) {
  list-style-type: decimal;
  padding-left: 1.25rem;
}
</style>
