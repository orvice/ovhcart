<template>
  <div>
    <v-card-title class="text-h5 py-4">Settings</v-card-title>
    <v-card-text>
      <v-form @submit.prevent="saveSettings">
        <v-text-field
          v-model="localApiToken"
          label="OVH API Token"
          type="password"
          hint="Your OVH API token will be stored locally"
          placeholder="Enter your OVH API token"
          persistent-hint
          :rules="[v => !!v || 'Token is required']"
          class="mb-4"
        ></v-text-field>

        <v-alert
          type="info"
          variant="tonal"
          class="mb-4"
          density="compact"
        >
          <div class="d-flex align-center">
            <span>Get your API token from:</span>
            <v-btn
              variant="text"
              color="info"
              href="https://eu.api.ovh.com/console/?section=%2Forder&branch=v1#get-/order/cart"
              target="_blank"
              class="ml-1"
              density="compact"
            >
              OVH API Console
              <v-icon icon="mdi-open-in-new" size="small" class="ml-1"></v-icon>
            </v-btn>
          </div>
        </v-alert>

        <v-select
          v-model="localSelectedSite"
          label="OVH Subsidiary for Cart Creation"
          :items="availableSites"
          hint="Select your OVH region for cart creation (ovhSubsidiary)"
          persistent-hint
          item-title="name"
          item-value="code"
          return-object
          class="mb-4"
        >
          <template v-slot:selection="{ item }">
            <div class="d-flex align-center">
              <v-avatar size="24" class="me-2">
                <v-img :src="getCountryFlag(item?.raw?.code)" alt="Country flag"></v-img>
              </v-avatar>
              {{ item?.raw?.name || 'Select site' }} {{ item?.raw?.code ? `(${item.raw.code})` : '' }}
            </div>
          </template>

          <template v-slot:item="{ item, props }">
            <v-list-item v-bind="props">
              <template v-slot:prepend>
                <v-avatar size="24">
                  <v-img :src="getCountryFlag(item?.code)" alt="Country flag"></v-img>
                </v-avatar>
              </template>
              <v-list-item-title>{{ item?.name || '-' }}</v-list-item-title>
              <v-list-item-subtitle>{{ item?.code || '' }}</v-list-item-subtitle>
            </v-list-item>
          </template>
        </v-select>

        <v-card variant="outlined" class="mb-4 pa-3">
          <v-card-title class="text-subtitle-1">Current Settings</v-card-title>
          <v-card-text>
            <p><strong>API Endpoint:</strong> https://eu.api.ovh.com/v1 (Fixed)</p>
            <p><strong>Selected Site:</strong> {{ localSelectedSite?.code || 'Not selected' }} - {{ localSelectedSite?.name || 'Not selected' }}</p>
            <p class="text-caption">Note: The API endpoint is fixed to eu.api.ovh.com regardless of site selection</p>
          </v-card-text>
        </v-card>

        <!-- Theme Selection -->
        <v-card variant="outlined" class="mb-4 pa-3">
          <v-card-title class="text-subtitle-1">
            <v-icon icon="mdi-theme-light-dark" class="mr-2"></v-icon>
            Theme Settings
          </v-card-title>
          <v-card-text>
            <v-radio-group
              v-model="selectedTheme"
              inline
              hide-details
            >
              <v-radio
                value="light"
                label="Light"
              >
                <template v-slot:label>
                  <div class="d-flex align-center">
                    <v-icon icon="mdi-white-balance-sunny" color="amber" class="mr-2"></v-icon>
                    Light
                  </div>
                </template>
              </v-radio>
              <v-radio
                value="dark"
                label="Dark"
              >
                <template v-slot:label>
                  <div class="d-flex align-center">
                    <v-icon icon="mdi-weather-night" color="blue-darken-3" class="mr-2"></v-icon>
                    Dark
                  </div>
                </template>
              </v-radio>
              <v-radio
                value="auto"
                label="Auto"
              >
                <template v-slot:label>
                  <div class="d-flex align-center">
                    <v-icon icon="mdi-theme-light-dark" color="grey" class="mr-2"></v-icon>
                    Auto (System)
                  </div>
                </template>
              </v-radio>
            </v-radio-group>
          </v-card-text>
        </v-card>

        <v-btn
          color="primary"
          type="submit"
          :loading="tokenSaving"
        >
          Save Settings
        </v-btn>
      </v-form>
    </v-card-text>

    <v-snackbar
      v-model="showSettingsSaved"
      :timeout="3000"
      color="success"
    >
      Settings saved successfully!
    </v-snackbar>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue'
import { availableSites } from '../config/sites'

const emit = defineEmits(['update:apiToken', 'update:selectedSite', 'settingsSaved'])

const props = defineProps({
  apiToken: {
    type: String,
    required: true
  },
  selectedSite: {
    type: Object,
    required: true
  }
})

// Use computed properties for two-way binding
const localApiToken = computed({
  get: () => props.apiToken,
  set: (value) => emit('update:apiToken', value)
})

const localSelectedSite = computed({
  get: () => props.selectedSite,
  set: (value) => emit('update:selectedSite', value)
})

// Theme selection
const selectedTheme = ref('auto')
const tokenSaving = ref(false)
const showSettingsSaved = ref(false)

// Function to get country flag for site code
const getCountryFlag = (siteCode) => {
  // Check if siteCode is defined
  if (!siteCode) return 'https://flagcdn.com/48x36/xx.png'

  // Convert site code to country code for flags
  const countryCode = siteCode === 'EU' ? 'eu' : siteCode.toLowerCase()
  return `https://flagcdn.com/48x36/${countryCode}.png`
}

// Function to apply theme
const applyTheme = (theme) => {
  if (theme === 'auto') {
    // Check system preference
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    document.documentElement.setAttribute('data-theme', prefersDark ? 'dark' : 'light')
  } else {
    // Apply specified theme
    document.documentElement.setAttribute('data-theme', theme)
  }
}

// Watch for theme changes to apply immediately
watch(selectedTheme, (newTheme) => {
  applyTheme(newTheme)
})

// Function to save settings
const saveSettings = async () => {
  try {
    tokenSaving.value = true

    // Save token
    localStorage.setItem('ovhApiToken', localApiToken.value)

    // Save selected site
    if (localSelectedSite.value) {
      localStorage.setItem('ovhSiteCode', localSelectedSite.value.code)
    }

    // Save selected theme
    localStorage.setItem('ovhTheme', selectedTheme.value)

    showSettingsSaved.value = true
    emit('settingsSaved')
  } catch (err) {
    console.error('Error saving settings:', err)
  } finally {
    tokenSaving.value = false
  }
}

// On mounted, load saved theme
onMounted(() => {
  // Load saved theme
  const savedTheme = localStorage.getItem('ovhTheme')
  if (savedTheme) {
    selectedTheme.value = savedTheme
    applyTheme(savedTheme)
  } else {
    // Default to auto theme
    applyTheme('auto')
  }
})
</script>
