<script setup lang="ts">
import { ref } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { supabase } from '@/lib/supabaseClient'

const router = useRouter()
const route = useRoute()

const email = ref('')
const password = ref('')
const loading = ref(false)
const error = ref<string | null>(null)

async function handleLogin() {
  loading.value = true
  error.value = null

  const { data, error: authError } = await supabase.auth.signInWithPassword({
    email: email.value.trim(),
    password: password.value,
  })

  if (authError || !data.session) {
    loading.value = false
    error.value = 'Login failed. Please check your details and try again.'
    return
  }

  alert('Welcome back.')
  const redirect = (route.query.redirect as string) || '/admin'
  router.push(redirect)
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-b from-sky-50 via-slate-50 to-white flex items-center justify-center px-4">
    <div class="w-full max-w-md">
      <div class="text-center mb-8">
        <p class="text-xs font-semibold tracking-[0.2em] text-primary-600 uppercase mb-2">GraceDaily Admin</p>
        <h1 class="text-2xl font-semibold text-slate-900 mb-1">Sign in to serve</h1>
        <p class="text-sm text-slate-500">Only authorized church staff may post new devotions.</p>
      </div>

      <form
        class="gd-card p-6 space-y-4"
        @submit.prevent="handleLogin"
      >
        <div class="space-y-1">
          <label class="block text-xs font-medium text-slate-700">Email</label>
          <input
            v-model="email"
            type="email"
            required
            class="block w-full rounded-lg border border-slate-200 bg-slate-50 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
            autocomplete="email"
          >
        </div>

        <div class="space-y-1">
          <label class="block text-xs font-medium text-slate-700">Password</label>
          <input
            v-model="password"
            type="password"
            required
            class="block w-full rounded-lg border border-slate-200 bg-slate-50 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500"
            autocomplete="current-password"
          >
        </div>

        <p v-if="error" class="text-xs text-red-600">{{ error }}</p>

        <button
          type="submit"
          class="gd-btn-primary w-full text-sm"
          :disabled="loading"
        >
          <span v-if="loading">Signing in...</span>
          <span v-else>Sign in</span>
        </button>
      </form>
    </div>
  </div>
</template>
