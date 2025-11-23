<script setup lang="ts">
import { useRouter } from 'vue-router'

const router = useRouter()

function goHome() {
  router.push({ name: 'home' })
}
</script>

<template>
  <div class="min-h-screen flex flex-col items-center justify-center px-4">
    <h1 class="text-3xl font-semibold text-slate-900 mb-2">Page not found</h1>
    <p class="text-sm text-slate-500 mb-6">
      The page you are looking for does not exist.
    </p>
    <button
      class="gd-btn-primary text-sm"
      type="button"
      @click="goHome"
    >
      Go back home
    </button>
  </div>
</template>
