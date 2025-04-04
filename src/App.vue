<template>
  <v-app :theme="theme">
    <v-main class="main-content">
      <router-view />
    </v-main>
  </v-app>
</template>

<script setup>
  // Import the custom theme
  import '@/assets/theme.css'
  import { ref, onMounted, onUnmounted } from 'vue'

  const theme = ref('light')

  // Function to update theme based on system preference or stored value
  const updateTheme = () => {
    const savedTheme = localStorage.getItem('ovhTheme')
    if (savedTheme === 'dark') {
      theme.value = 'dark'
      document.documentElement.setAttribute('data-theme', 'dark')
    } else if (savedTheme === 'light') {
      theme.value = 'light'
      document.documentElement.setAttribute('data-theme', 'light')
    } else {
      // Auto mode - follow system preference
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
      theme.value = prefersDark ? 'dark' : 'light'
      document.documentElement.setAttribute('data-theme', prefersDark ? 'dark' : 'light')
    }
  }

  // Watch for system color scheme changes
  onMounted(() => {
    updateTheme()
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', updateTheme)
  })

  // Clean up event listener
  onUnmounted(() => {
    window.matchMedia('(prefers-color-scheme: dark)').removeEventListener('change', updateTheme)
  })
</script>

<style>
.main-content {
  background-color: var(--background-color);
  min-height: 100vh;
}
</style>
