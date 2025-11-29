<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'
import { supabase } from '@/lib/supabaseClient'
import { useIntersectionObserver } from '@vueuse/core'

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
const loadingMore = ref(false)
const error = ref<string | null>(null)
const selectedDevotion = ref<Devotion | null>(null)
const pageSize = 9
const hasMore = ref(true)
const loadMoreTrigger = ref(null)
const bookmarked = ref<Set<number>>(new Set())

// Today's devotion (most recent)
const todaysDevotion = computed(() => devotions.value[0] || null)

// Format & preview utilities
function formatDate(value: string) {
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return value
  const options: Intl.DateTimeFormatOptions = { 
    weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' 
  }
  return date.toLocaleDateString(undefined, options)
}

function getPlainText(html: string) {
  if (typeof document === 'undefined') {
    return html.replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim()
  }
  const div = document.createElement('div')
  div.innerHTML = html
  return (div.textContent || div.innerText || '').replace(/\s+/g, ' ').trim()
}

function getPreview(devotion: Devotion, length = 120) {
  const text = getPlainText(devotion.content)
  return text.length <= length ? text : text.slice(0, length) + '…'
}

function getVerseRefDisplay(verse: string) {
  const [ref] = verse.split('|')
  return (ref || '').trim()
}

function getVerseTooltipText(verse: string) {
  const parts = verse.split('|')
  if (parts.length > 1) {
    return parts.slice(1).join('|').trim()
  }
  return verse.trim()
}

// Data loading
async function loadDevotions(initial = false) {
  if (initial) {
    loading.value = true
    devotions.value = []
    hasMore.value = true
  } else loadingMore.value = true

  const from = initial ? 0 : devotions.value.length
  const to = from + pageSize - 1

  const { data, error: err } = await supabase
    .from('devotions')
    .select('*')
    .order('createdAt', { ascending: false })
    .range(from, to)

  if (err) {
    error.value = 'Unable to load devotions. Please try again later.'
  } else {
    const items = data as Devotion[]
    devotions.value = initial ? items : [...devotions.value, ...items]
    if (items.length < pageSize) hasMore.value = false
  }

  loading.value = false
  loadingMore.value = false
}

// Infinite scroll with IntersectionObserver
useIntersectionObserver(loadMoreTrigger, ([entry]) => {
  if (entry?.isIntersecting && hasMore.value && !loadingMore.value && !loading.value) {
    loadDevotions(false)
  }
})

// Modal & sharing
function openDevotion(devotion: Devotion) {
  selectedDevotion.value = devotion
  document.body.style.overflow = 'hidden'
}

function closeDevotion() {
  selectedDevotion.value = null
  document.body.style.overflow = ''
}

async function shareDevotion(devotion: Devotion) {
  const text = `"${devotion.title}"\n${formatDate(devotion.date)}\n\n${devotion.verse}\n\n${getPlainText(devotion.content)}\n\n— GraceDaily Devotional`

  if (navigator.share) {
    await navigator.share({ title: devotion.title, text })
  } else if (navigator.clipboard) {
    await navigator.clipboard.writeText(text)
    alert('Devotion copied to clipboard!')
  }
}

function toggleBookmark(id: number) {
  if (bookmarked.value.has(id)) {
    bookmarked.value.delete(id)
  } else {
    bookmarked.value.add(id)
  }
}

onMounted(() => loadDevotions(true))
</script>

<template>
  <div class="min-h-screen bg-gradient-to-b from-sky-50/30 via-white to-white">
    <!-- Hero: Today's Devotion -->
    <section v-if="todaysDevotion && !loading" class="gd-section pb-12">
      <div 
        class="gd-card overflow-hidden bg-gradient-to-br from-sky-100 via-blue-50 to-white border border-sky-200 shadow-lg shadow-sky-100/50 
               transform transition-all duration-1000 hover:scale-[1.01] hover:shadow-xl"
      >
        <div class="px-8 py-12 sm:px-12 lg:px-16">
          <div class="flex items-center gap-3 mb-4">
            <span class="text-xs font-semibold tracking-widest text-primary-600 uppercase">Today's Grace</span>
            <div class="h-px flex-1 bg-gradient-to-r from-primary-300 to-transparent"></div>
          </div>

          <h1 class="text-3xl sm:text-4xl lg:text-5xl font-bold text-slate-900 mb-4 leading-tight">
            {{ todaysDevotion.title }}
          </h1>

          <p class="text-lg italic text-sky-700 mb-6 font-medium">
            <abbr :title="getVerseTooltipText(todaysDevotion.verse)" class="cursor-help">
              {{ getVerseRefDisplay(todaysDevotion.verse) }}
            </abbr>
          </p>

          <p class="text-base text-slate-700 max-w-4xl leading-relaxed mb-8">
            {{ getPreview(todaysDevotion, 280) }}
          </p>

          <div class="flex flex-wrap gap-4">
            <button 
              @click="openDevotion(todaysDevotion)"
              class="gd-btn-primary px-6 py-3 text-sm font-medium shadow-md hover:shadow-lg transform hover:-translate-y-0.5 transition-all"
            >
              Read Today's Devotion
            </button>
            <button 
              @click="shareDevotion(todaysDevotion)"
              class="px-5 py-3 text-sm font-medium text-slate-600 border border-slate-300 rounded-full hover:bg-slate-50 transition"
            >
              Share with someone
            </button>
          </div>
        </div>
      </div>
    </section>

    <!-- Previous Devotions Grid -->
    <section class="gd-section">
      <div class="flex items-center justify-between mb-8">
        <h2 class="text-2xl font-bold text-slate-900">Previous Devotions</h2>
        <p class="text-sm text-slate-500" v-if="devotions.length > 1">
          {{ devotions.length - 1 }} more devotion{{ devotions.length > 2 ? 's' : '' }} available
        </p>
      </div>

      <div v-if="loading && devotions.length === 0" class="text-center py-12">
        <div class="inline-block animate-spin rounded-full h-10 w-10 border-4 border-primary-200 border-t-primary-600"></div>
        <p class="mt-4 text-slate-600">Preparing your daily bread…</p>
      </div>

      <div v-else-if="error" class="text-center py-12 text-red-600">{{ error }}</div>

      <div class="grid gap-6 sm:grid-cols-2 lg:grid-cols-3">
        <article
          v-for="devotion in devotions.slice(1)"
          :key="devotion.id"
          class="gd-card group relative overflow-hidden rounded-2xl border border-slate-100 bg-white/70 backdrop-blur 
                 shadow-md hover:shadow-2xl transition-all duration-500 hover:-translate-y-2 cursor-pointer"
          @click="openDevotion(devotion)"
        >
          <!-- Subtle gradient overlay on hover -->
          <div class="absolute inset-0 bg-gradient-to-t from-primary-600/5 to-transparent opacity-0 group-hover:opacity-100 transition-opacity"></div>

          <div class="p-6 relative z-10">
            <div class="flex justify-between items-start mb-3">
              <span class="text-xs font-medium text-primary-600">
                {{ formatDate(devotion.date).split(',')[0] }}
              </span>
              <button 
                @click.stop="toggleBookmark(devotion.id)"
                class="text-slate-400 hover:text-amber-600 transition"
              >
                <svg :class="{ 'fill-amber-600': bookmarked.has(devotion.id) }" 
                     class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                        d="M5 5a2 2 0 012-2h10a2 2 0 012 2v16l-7-3.5L5 21V5z" />
                </svg>
              </button>
            </div>

            <h3 class="font-semibold text-slate-900 mb-2 line-clamp-2 group-hover:text-primary-700 transition">
              {{ devotion.title }}
            </h3>
            <p class="text-xs italic text-slate-600 mb-3 line-clamp-1">
              <abbr :title="getVerseTooltipText(devotion.verse)" class="cursor-help">
                {{ getVerseRefDisplay(devotion.verse) }}
              </abbr>
            </p>
            <p class="text-sm text-slate-600 line-clamp-3">
              {{ getPreview(devotion, 100) }}
            </p>

            <div class="mt-5 flex justify-between items-center opacity-0 group-hover:opacity-100 transition-opacity">
              <span class="text-xs font-medium text-primary-700">Read →</span>
              <button @click.stop="shareDevotion(devotion)" class="text-xs text-slate-500 hover:text-slate-700">
                Share
              </button>
            </div>
          </div>
        </article>
      </div>

      <!-- Infinite scroll trigger -->
      <div v-if="hasMore" ref="loadMoreTrigger" class="py-10 text-center">
        <span class="text-sm text-slate-500 animate-pulse">Loading more grace…</span>
      </div>
    </section>

    <!-- Full Devotion Modal -->
    <Teleport to="body">
      <div 
        v-if="selectedDevotion" 
        class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/70 backdrop-blur-sm"
        @click.self="closeDevotion"
      >
        <div 
          class="gd-card max-w-2xl w-full max-h-[92vh] overflow-y-auto bg-white/95 backdrop-blur-lg rounded-3xl shadow-2xl 
                 border border-sky-100 animate-in fade-in zoom-in-95 duration-300"
        >
          <!-- Header -->
          <div class="sticky top-0 bg-white/90 backdrop-blur border-b border-sky-100 px-6 py-5 flex justify-between items-start">
            <div>
              <p class="text-xs font-semibold text-primary-600 uppercase tracking-wider">
                {{ formatDate(selectedDevotion.date) }}
              </p>
              <h2 class="text-2xl font-bold text-slate-900 mt-1">
                {{ selectedDevotion.title }}
              </h2>
            </div>
            <button @click="closeDevotion" class="text-slate-400 hover:text-slate-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>

          <!-- Verse Highlight -->
          <div class="mx-6 mt-6 p-5 bg-gradient-to-r from-sky-50 to-blue-50 rounded-2xl border border-sky-200">
            <p class="text-xs font-bold uppercase tracking-widest text-sky-700 mb-2">Memory Verse</p>
            <p class="text-lg italic font-medium text-sky-900 leading-relaxed">
              <abbr :title="getVerseTooltipText(selectedDevotion.verse)" class="cursor-help">
                {{ getVerseRefDisplay(selectedDevotion.verse) }}
              </abbr>
            </p>
          </div>

          <!-- Content -->
          <div class="px-6 py-8 prose prose-sm max-w-none text-slate-700 leading-relaxed">
            <div v-html="selectedDevotion.content"></div>
          </div>

          <!-- Footer Actions -->
          <div class="sticky bottom-0 bg-white/90 backdrop-blur border-t border-sky-100 px-6 py-5 flex justify-between items-center">
            <button 
              @click="toggleBookmark(selectedDevotion.id)"
              class="flex items-center gap-2 text-sm text-slate-600 hover:text-amber-700 transition"
            >
              <svg :class="{ 'fill-amber-600': bookmarked.has(selectedDevotion.id) }" 
                   class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                      d="M5 5a2 2 0 012-2h10a2 2 0 012 2v16l-7-3.5L5 21V5z" />
              </svg>
              <span>{{ bookmarked.has(selectedDevotion.id) ? 'Saved' : 'Save for later' }}</span>
            </button>

            <button 
              @click="shareDevotion(selectedDevotion)"
              class="gd-btn-primary px-5 py-2.5 text-sm font-medium"
            >
              Share This Devotion
            </button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.animate-in { animation: fadeIn 0.4s ease-out; }
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.prose :deep(p) { margin-bottom: 1.1rem; }
.prose :deep(strong) { color: #1e293b; font-weight: 600; }
.prose :deep(ul), .prose :deep(ol) { padding-left: 1.5rem; margin: 1rem 0; }
</style>