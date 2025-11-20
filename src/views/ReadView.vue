<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { supabase } from '@/lib/supabaseClient'

interface Devotion {
  id: number
  title: string
  verse: string
  content: string
  date: string
  createdAt: string
}

const devotions = ref<Devotion[]>([])
const loading = ref(true)
const error = ref<string | null>(null)
const selectedDevotion = ref<Devotion | null>(null)

function formatDate(value: string) {
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return value
  return date.toLocaleDateString(undefined, { year: 'numeric', month: 'short', day: 'numeric' })
}

async function loadDevotions() {
  loading.value = true
  error.value = null
  const { data, error: queryError } = await supabase
    .from('devotions')
    .select('*')
    .order('date', { ascending: false })

  if (queryError) {
    error.value = 'Unable to load devotions right now.'
    loading.value = false
    return
  }

  devotions.value = (data ?? []) as Devotion[]
  loading.value = false
}

function openDevotion(devotion: Devotion) {
  selectedDevotion.value = devotion
}

function closeDevotion() {
  selectedDevotion.value = null
}

async function shareDevotion(devotion: Devotion) {
  const text = `${devotion.title} (GraceDaily)\n${formatDate(devotion.date)}\n\nMemory verse: ${devotion.verse}\n\n${devotion.content}`

  try {
    if (navigator.share) {
      await navigator.share({
        title: devotion.title,
        text,
      })
      return
    }

    await navigator.clipboard.writeText(text)
    alert('Devotion copied to your clipboard.')
  } catch (e) {
    alert('Unable to share this devotion. You can copy it manually.')
  }
}

onMounted(loadDevotions)
</script>

<template>
  <div class="py-8 sm:py-12 lg:py-16">
    <section class="gd-section mb-10">
      <div class="gd-card overflow-hidden bg-gradient-to-br from-sky-50 to-white">
        <div class="px-6 py-8 sm:px-8 sm:py-10">
          <p class="text-xs font-semibold tracking-[0.2em] text-primary-600 uppercase mb-2">GraceDaily Devotional</p>
          <h1 class="text-3xl sm:text-4xl font-semibold text-slate-900 mb-3">Daily Bread for the Soul</h1>
          <p class="text-sm sm:text-base text-slate-600 max-w-2xl">
            Be still and receive today's word. Each devotion is crafted to encourage, challenge, and draw you
            closer to Jesus in the middle of everyday life.
          </p>
        </div>
      </div>
    </section>

    <section class="gd-section">
      <div class="flex items-center justify-between mb-4">
        <h2 class="text-lg font-semibold text-slate-900">Latest devotions</h2>
        <p class="text-xs text-slate-500" v-if="!loading && devotions.length">
          Showing {{ devotions.length }} devotion{{ devotions.length === 1 ? '' : 's' }}
        </p>
      </div>

      <div v-if="loading" class="text-sm text-slate-500">Loading devotions...</div>
      <div v-else-if="error" class="text-sm text-red-600">{{ error }}</div>
      <div v-else-if="!devotions.length" class="text-sm text-slate-500">
        No devotions have been published yet. Please check back later.
      </div>

      <div
        v-else
        class="grid gap-4 sm:gap-6 sm:grid-cols-2 lg:grid-cols-3 mt-4"
      >
        <article
          v-for="devotion in devotions"
          :key="devotion.id"
          class="gd-card p-5 flex flex-col justify-between cursor-pointer hover:shadow-lg transition-shadow"
          @click="openDevotion(devotion)"
        >
          <div>
            <p class="text-xs text-slate-500 mb-1">{{ formatDate(devotion.date) }}</p>
            <h3 class="text-base font-semibold text-slate-900 mb-2 line-clamp-2">
              {{ devotion.title }}
            </h3>
            <p class="text-xs text-slate-600 mb-3 italic line-clamp-2">
              {{ devotion.verse }}
            </p>
            <p class="text-sm text-slate-600 line-clamp-3">
              {{ devotion.content.slice(0, 100) }}<span v-if="devotion.content.length > 100">26hellip;</span>
            </p>
          </div>
          <div class="mt-4 flex justify-between items-center">
            <button
              type="button"
              class="text-xs font-medium text-primary-700 hover:text-primary-800"
            >
              Read more
            </button>
            <button
              type="button"
              class="text-[11px] text-slate-500 hover:text-slate-700"
              @click.stop="shareDevotion(devotion)"
            >
              Share
            </button>
          </div>
        </article>
      </div>
    </section>

    <div
      v-if="selectedDevotion"
      class="fixed inset-0 z-40 flex items-center justify-center bg-slate-900/50 px-4"
    >
      <div class="gd-card max-w-2xl w-full max-h-[90vh] overflow-y-auto">
        <div class="flex justify-between items-start border-b border-slate-100 px-6 py-4">
          <div>
            <p class="text-xs text-slate-500 mb-1">
              {{ formatDate(selectedDevotion.date) }}
            </p>
            <h2 class="text-xl font-semibold text-slate-900">
              {{ selectedDevotion.title }}
            </h2>
          </div>
          <button
            type="button"
            class="text-xs text-slate-400 hover:text-slate-600"
            @click="closeDevotion"
          >
            Close
          </button>
        </div>

        <div class="px-6 py-4 space-y-4">
          <div class="bg-sky-50 border border-sky-100 rounded-lg px-4 py-3 text-sm text-sky-900 italic">
            <p class="font-medium text-xs uppercase tracking-[0.18em] text-sky-700 mb-1">Memory verse</p>
            <p>
              {{ selectedDevotion.verse }}
            </p>
          </div>

          <div class="prose prose-sm max-w-none text-slate-700 whitespace-pre-line">
            {{ selectedDevotion.content }}
          </div>
        </div>

        <div class="px-6 py-4 border-t border-slate-100 flex justify-end gap-2">
          <button
            type="button"
            class="text-xs text-slate-500 hover:text-slate-700"
            @click="closeDevotion"
          >
            Close
          </button>
          <button
            type="button"
            class="gd-btn-primary text-xs"
            @click="shareDevotion(selectedDevotion)"
          >
            Share devotion
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
