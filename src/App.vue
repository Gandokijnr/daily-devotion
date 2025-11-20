<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="min-h-screen flex flex-col bg-gradient-to-b from-sky-50 via-slate-50 to-white">
    <header class="border-b border-slate-200/70 bg-white/80 backdrop-blur-sm">
      <div class="gd-section flex items-center justify-between py-3">
        <div>
          <p class="text-[10px] font-semibold tracking-[0.25em] text-primary-600 uppercase mb-0.5">
            GraceDaily
          </p>
          <p class="text-xs text-slate-500">Daily devotionals for the local church</p>
        </div>
        <nav class="flex items-center gap-3 text-xs text-slate-600">
          <RouterLink
            to="/"
            class="hover:text-primary-700 transition"
          >
            Today's reading
          </RouterLink>
          <!-- <RouterLink
            to="/admin/login"
            class="hover:text-primary-700 transition"
          >
            Admin
          </RouterLink> -->
        </nav>
      </div>
    </header>

    <main class="flex-1">
      <RouterView />
    </main>

    <footer class="border-t border-slate-200/70 bg-white/70 mt-8">
      <div class="gd-section py-3 flex justify-between items-center text-[11px] text-slate-400">
        <p>GraceDaily — a quiet space to meet with God each day.</p>
        <p>“The word of our God stands forever.” — Isaiah 40:8</p>
      </div>
    </footer>
  </div>
</template>

<style scoped></style>
