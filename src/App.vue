<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  // Auto dark mode based on system preference
  const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
  const applyTheme = (e: MediaQueryList | MediaQueryListEvent) => {
    document.documentElement.classList.toggle('dark', e.matches)
  }
  applyTheme(mediaQuery)
  mediaQuery.addEventListener('change', applyTheme)
})
</script>

<template>
  <router-view />
</template>
