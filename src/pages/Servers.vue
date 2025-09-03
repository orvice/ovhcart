<template>
  <v-container>
    <v-card>
      <v-card-title class="text-h5 py-4 d-flex align-center">
        Server Plans
        <v-chip 
          class="ml-2" 
          color="primary" 
          size="small"
        >
          Subsidiary: {{ selectedSite?.code || 'IE' }}
        </v-chip>
        <v-spacer />
        <div class="d-flex flex-column align-end mr-4">
          <div 
            v-if="lastRefreshTime"
            class="text-caption text-right mb-1"
          >
            Last updated: {{ formatRefreshTime(lastRefreshTime) }}
            <v-chip 
              size="x-small" 
              :color="isFromCache ? 'amber' : 'success'" 
              class="ml-2"
            >
              {{ isFromCache ? 'Cache' : 'API' }}
            </v-chip>
          </div>
        </div>
        <v-btn
          color="primary"
          variant="elevated"
          prepend-icon="mdi-refresh"
          :loading="loading"
          @click="forceRefreshServers"
        >
          Refresh Servers
        </v-btn>
      </v-card-title>
      
      <v-card-text>
        <div v-if="loading">
          <v-progress-circular
            indeterminate
            color="primary"
          />
          <span class="ml-2">Loading server catalog...</span>
        </div>
        
        <v-alert
          v-else-if="serverError"
          type="error"
          title="Error"
          :text="serverError.message || 'Failed to load server catalog'"
        />
        
        <v-alert
          v-else-if="!serverCatalog || !serverCatalog.plans || serverCatalog.plans.length === 0"
          type="info"
          text="No server plans available"
        />
        
        <div v-else>
          <!-- Price Filter Section -->
          <v-card
            variant="outlined"
            class="mb-4 pa-4"
          >
            <v-card-title class="text-subtitle-1">
              <v-icon
                icon="mdi-filter-variant"
                color="primary"
                class="mr-2"
              />
              Filter by Price Range
            </v-card-title>
            <v-card-text>
              <v-row align="center">
                <v-col cols="12" md="4">
                  <v-text-field
                    v-model.number="priceFilter.min"
                    label="Min Price (€)"
                    type="number"
                    min="0"
                    density="compact"
                    variant="outlined"
                    prepend-inner-icon="mdi-currency-eur"
                    @input="filterServersByPrice"
                  />
                </v-col>
                <v-col cols="12" md="4">
                  <v-text-field
                    v-model.number="priceFilter.max"
                    label="Max Price (€)"
                    type="number"
                    min="0"
                    density="compact"
                    variant="outlined"
                    prepend-inner-icon="mdi-currency-eur"
                    @input="filterServersByPrice"
                  />
                </v-col>
                <v-col cols="12" md="4">
                  <v-btn
                    color="primary"
                    variant="tonal"
                    @click="clearPriceFilter"
                    block
                  >
                    <v-icon icon="mdi-filter-off" class="mr-2" />
                    Clear Filter
                  </v-btn>
                </v-col>
              </v-row>
              <div v-if="filteredPlansCount !== null" class="text-caption mt-2">
                Showing {{ filteredPlansCount }} of {{ totalPlansCount }} server plans
              </div>
            </v-card-text>
          </v-card>
          
          <!-- Manual Plan Code Input Button -->
          <v-card 
            variant="outlined" 
            class="mb-4 pa-4"
          >
            <v-card-title class="text-subtitle-1">
              <v-icon 
                icon="mdi-keyboard" 
                color="primary" 
                class="mr-2" 
              />
              Manually Add Server by Plan Code
            </v-card-title>
            <v-card-text>
              <p class="text-body-2 mb-4">
                If you know the exact OVH plan code for a server, you can manually add it to your cart.
              </p>
              <v-btn 
                color="primary" 
                variant="elevated" 
                prepend-icon="mdi-plus"
                @click="showManualPlanCodeDialog = true"
              >
                Enter Plan Code
              </v-btn>
            </v-card-text>
          </v-card>
          
          <v-row>
            <v-col
              v-for="(plan, index) in filteredPlans"
              :key="index"
              cols="12"
              md="4"
            >
              <v-card class="h-100">
                <v-card-title class="text-h6">
                  {{ plan.planCode || 'Server Plan' }}
                </v-card-title>
                <v-card-subtitle 
                  v-if="plan.invoiceName" 
                  class="text-subtitle-1 font-weight-bold"
                >
                  {{ plan.invoiceName }}
                </v-card-subtitle>
                <v-card-subtitle v-else-if="plan.description">
                  {{ plan.description }}
                </v-card-subtitle>
                <v-card-text>
                  <v-list density="compact">
                    <!-- Family info -->
                    <v-list-item v-if="plan.family">
                      <template #prepend>
                        <v-icon 
                          icon="mdi-server" 
                          color="primary" 
                        />
                      </template>
                      <v-list-item-title>Family</v-list-item-title>
                      <v-list-item-subtitle>{{ plan.family }}</v-list-item-subtitle>
                    </v-list-item>
                    
                    <!-- Product type -->
                    <v-list-item v-if="plan.productType">
                      <template #prepend>
                        <v-icon 
                          icon="mdi-tag" 
                          color="primary" 
                        />
                      </template>
                      <v-list-item-title>Type</v-list-item-title>
                      <v-list-item-subtitle>{{ plan.productType }}</v-list-item-subtitle>
                    </v-list-item>

                    <!-- Memory info -->
                    <v-list-item v-if="getAddonInfo(plan, 'memory')">
                      <template #prepend>
                        <v-icon 
                          icon="mdi-memory" 
                          color="green" 
                        />
                      </template>
                      <v-list-item-title>Memory</v-list-item-title>
                      <v-list-item-subtitle>{{ getAddonInfo(plan, 'memory') }}</v-list-item-subtitle>
                    </v-list-item>

                    <!-- Storage info -->
                    <v-list-item v-if="getAddonInfo(plan, 'storage')">
                      <template #prepend>
                        <v-icon 
                          icon="mdi-harddisk" 
                          color="amber" 
                        />
                      </template>
                      <v-list-item-title>Storage</v-list-item-title>
                      <v-list-item-subtitle>{{ getAddonInfo(plan, 'storage') }}</v-list-item-subtitle>
                    </v-list-item>
                    
                    <!-- Bandwidth info -->
                    <v-list-item v-if="getAddonInfo(plan, 'bandwidth')">
                      <template #prepend>
                        <v-icon 
                          icon="mdi-speedometer" 
                          color="purple" 
                        />
                      </template>
                      <v-list-item-title>Bandwidth</v-list-item-title>
                      <v-list-item-subtitle>{{ getAddonInfo(plan, 'bandwidth') }}</v-list-item-subtitle>
                    </v-list-item>
                    
                    <!-- Price info -->
                    <!-- Price info -->
                    <v-list-item
                      v-for="(pricing, priceIndex) in plan.pricings || []"
                      :key="priceIndex"
                    >
                      <template #prepend>
                        <v-icon icon="mdi-currency-eur" color="primary" />
                      </template>
                      <v-list-item-title>{{ pricing.description }}</v-list-item-title>
                      <v-list-item-subtitle>{{ formatPricing(pricing) }}</v-list-item-subtitle>
                    </v-list-item>
                  </v-list>
                  
                  <v-chip-group 
                          v-if="plan.properties && plan.properties.length > 0"
                          class="mt-3"
                        >
                    <v-chip
                      v-for="(prop, propIndex) in plan.properties"
                      :key="propIndex"
                      size="small"
                      variant="outlined"
                      :color="getPropertyColor(prop.name)"
                    >
                      {{ prop.name }}: {{ prop.value }}
                    </v-chip>
                  </v-chip-group>
                  
                  <!-- Configuration options -->
                  <div v-if="plan.configurations && plan.configurations.length > 0" class="mt-4">
                    <v-list density="compact">
                      <v-list-subheader>Available Configurations</v-list-subheader>
                      
                      <v-list-item
                        v-for="(config, configIndex) in plan.configurations" 
                        :key="configIndex"
                      >
                        <template v-slot:prepend>
                          <v-chip
                            size="small"
                            :color="getPropertyColor(config.name)"
                            class="mr-2"
                          >
                            {{ formatAddonCode(config.name) }}
                          </v-chip>
                        </template>
                        
                        <v-list-item-title>{{ config.name }}</v-list-item-title>
                        
                        <v-list-item-subtitle>
                          <span class="text-caption">{{ config.isMandatory ? '(Required)' : '(Optional)' }}</span>
                          <div class="mt-1 text-caption" v-if="config.values && config.values.length > 0">
                            Options: {{ config.values.join(', ') }}
                          </div>
                        </v-list-item-subtitle>
                      </v-list-item>
                    </v-list>
                  </div>
                </v-card-text>
                <v-card-actions>
                  <v-btn
                    color="primary"
                    variant="text"
                    :loading="plan.addingToCart"
                    :disabled="!props.activeCartId || !plan.planCode"
                    @click="addServerToCart(plan)"
                  >
                    Add to Cart
                  </v-btn>
                  <v-btn
                    color="primary"
                    variant="text"
                    @click="showServerDetails(plan)"
                  >
                    View Details
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-col>
          </v-row>
        </div>
      </v-card-text>
    </v-card>

    <!-- Manual Plan Code Dialog -->
    <v-dialog
      v-model="showManualPlanCodeDialog"
      max-width="500px"
    >
      <v-card>
        <v-card-title class="text-h5">
            Add Server by Plan Code
          </v-card-title>
        <v-card-text>
          <v-form @submit.prevent="addServerByPlanCode">
            <v-text-field
              v-model="manualPlanCode"
              label="Plan Code"
              hint="Enter the exact OVH plan code"
              persistent-hint
              :rules="[v => !!v || 'Plan code is required']"
              required
              class="mb-4"
            />
            
            <v-select
              v-model="manualDuration"
              :items="durations"
              item-title="text"
              item-value="value"
              label="Duration"
              hint="Select contract duration"
              persistent-hint
              class="mb-4"
            />
            
            <v-text-field
              v-model="manualQuantity"
              label="Quantity"
              type="number"
              min="1"
              hint="Number of items to add"
              persistent-hint
              :rules="[v => !!v && v > 0 || 'Quantity must be at least 1']"
              required
            />
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="error"
            variant="text"
            @click="showManualPlanCodeDialog = false"
          >
            Cancel
          </v-btn>
          <v-btn
            color="success"
            variant="elevated"
            :loading="addingManualPlan"
            @click="addServerByPlanCode"
          >
            Add to Cart
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Server Details Dialog -->
    <v-dialog
      v-model="showServerDetailsDialog"
      max-width="600px"
    >
      <v-card v-if="selectedServer">
        <v-card-title class="text-h5">
          {{ selectedServer.planCode }}
          <v-chip
            v-if="selectedServer.family"
            size="small"
            color="primary"
            class="ml-2"
          >
            {{ selectedServer.family }}
          </v-chip>
        </v-card-title>
        <v-card-subtitle v-if="selectedServer.description">
          {{ selectedServer.description }}
        </v-card-subtitle>
        <v-card-text>
          <v-list density="compact">
            <v-list-item 
              v-for="(prop, index) in selectedServer.properties" 
              :key="index"
            >
              <v-list-item-title>{{ prop.name }}</v-list-item-title>
              <v-list-item-subtitle>{{ prop.value }}</v-list-item-subtitle>
            </v-list-item>
            <v-list-item v-if="getPlanPrice(selectedServer)">
              <v-list-item-title>Price</v-list-item-title>
              <v-list-item-subtitle>{{ getPlanPrice(selectedServer) }}</v-list-item-subtitle>
            </v-list-item>
          </v-list>

          <v-divider class="my-4" />
          
          <!-- Duration selection -->
          <v-select
            v-model="selectedDuration"
            :items="durations"
            item-title="text"
            item-value="value"
            label="Contract Duration"
            hint="Select your preferred contract duration"
            persistent-hint
            class="mb-4"
          />
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn
            color="primary"
            variant="text"
            @click="showServerDetailsDialog = false"
          >
            Close
          </v-btn>
          <v-btn
            color="success"
            variant="elevated"
            :loading="selectedServer.addingToCart"
            :disabled="!activeCartId || !selectedServer.planCode"
            @click="addSelectedServerToCart"
          >
            Add to Cart
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script setup>
import { ref, onMounted, watch, computed } from 'vue';
import { useRouter } from 'vue-router';

// Props from parent component
const props = defineProps({
  apiToken: {
    type: String,
    required: false,
    default: ''
  },
  selectedSite: {
    type: Object,
    default: () => ({ code: 'IE' })
  },
  activeCartId: {
    type: String,
    default: ''
  }
});

// Emit events to parent
const emit = defineEmits([
  'refresh-cart-details',
  'show-success',
  'show-error'
]);

// State
const loading = ref(false);
const serverError = ref(null);
const serverCatalog = ref({ plans: [] });
const showManualPlanCodeDialog = ref(false);
const addingManualPlan = ref(false);
const manualPlanCode = ref('');
const manualDuration = ref('P1M');
const manualQuantity = ref(1);
const showServerDetailsDialog = ref(false);
const selectedServer = ref(null);
const selectedDuration = ref('P1M');
const lastRefreshTime = ref(null); // 最后刷新时间
const isFromCache = ref(false); // 数据是否来自缓存

// Price filter state
const priceFilter = ref({
  min: null,
  max: null
});

// Durations list for selection
const durations = [
  { text: '1 Month', value: 'P1M' },
  { text: '3 Months', value: 'P3M' },
  { text: '6 Months', value: 'P6M' },
  { text: '12 Months', value: 'P1Y' },
  { text: '24 Months', value: 'P2Y' }
];

// Function to get API endpoint
const getApiEndpoint = () => {
  return 'https://eu.api.ovh.com/v1';
};

// Function to extract addon family information
const getAddonInfo = (plan, familyName) => {
  if (!plan.addonFamilies || plan.addonFamilies.length === 0) {
    return null;
  }

  const family = plan.addonFamilies.find(f => f.name === familyName);
  if (!family) {
    return null;
  }

  // Return default addon if available, otherwise first addon
  const addonCode = family.default || (family.addons && family.addons.length > 0 ? family.addons[0] : null);
  if (!addonCode) {
    return null;
  }

  // Format the addon code for display
  return formatAddonCode(addonCode);
};

// Format addon codes to be more readable
const formatAddonCode = (code) => {
  // Replace hyphens with spaces
  let formatted = code.replace(/-/g, ' ');
  
  // Extract specific information based on patterns in the code
  if (code.includes('ram-')) {
    // Examples: ram-128g-ecc-2666-24sys -> 128GB ECC RAM
    const match = code.match(/ram-(\d+)g/);
    if (match && match[1]) {
      return `${match[1]}GB RAM`;
    }
  } else if (code.includes('bandwidth-')) {
    // Examples: bandwidth-500-24sys -> 500 Mbps
    const match = code.match(/bandwidth-(\d+)/);
    if (match && match[1]) {
      return `${match[1]} Mbps`;
    }
  } else if (code.includes('raid') && code.includes('nvme')) {
    // Extract storage size for NVMe drives
    const match = code.match(/-(\d+)nvme/);
    if (match && match[1]) {
      return `${match[1]}GB NVMe SSD`;
    }
  } else if (code.includes('raid') && code.includes('sa')) {
    // Extract storage size for SATA drives
    const match = code.match(/-(\d+)sa/);
    if (match && match[1]) {
      return `${match[1]}GB SATA Drive`;
    }
  }
  
  // If no specific pattern matched, return the cleaned up code
  return formatted;
};

// Function to get plan price
const getPlanPrice = (plan) => {
  if (!plan || !Array.isArray(plan.pricings) || plan.pricings.length === 0) {
    return 'Price not available';
  }
  
  const pricing = plan.pricings[0];
  
  // Price is a numeric value in the sample data (in micro units, need to convert to standard currency)
  if (typeof pricing.price === 'number') {
    // Convert from micro units (e.g., 2999000000) to standard currency (e.g., 29.99)
    const priceValue = (pricing.price / 100000000).toFixed(2);
    // Use EUR as default currency or customize based on your needs
    return `€${priceValue}`;
  }
  // Handle legacy format if present
  else if (pricing.price && typeof pricing.price === 'object') {
    return `${pricing.price.text || `${pricing.price.value} ${pricing.price.currencyCode || 'EUR'}`}`;
  }
  
  return 'Price not available';
};

// Function to get numeric price value for filtering
const getNumericPrice = (plan) => {
  if (!plan || !Array.isArray(plan.pricings) || plan.pricings.length === 0) {
    return null;
  }
  
  const pricing = plan.pricings.length > 1 ? plan.pricings[1] : plan.pricings[0];
  
  if (typeof pricing.price === 'number') {
    // Convert from micro units to standard currency
    return pricing.price / 100000000;
  }
  else if (pricing.price && typeof pricing.price === 'object' && pricing.price.value) {
    return parseFloat(pricing.price.value);
  }
  
  return null;
};
// Format a single pricing entry for display
const formatPricing = (pricing) => {
  let amountText = 'Price not available';
  if (typeof pricing.price === 'number') {
    const priceValue = (pricing.price / 100000000).toFixed(2);
    amountText = `€${priceValue}`;
  } else if (pricing.price && typeof pricing.price === 'object') {
    amountText = pricing.price.text || `${pricing.price.value} ${pricing.price.currencyCode || 'EUR'}`;
  }
  return `${pricing.description}: ${amountText}`;
};

// Computed property for filtered plans
const filteredPlans = computed(() => {
  if (!serverCatalog.value.plans) {
    return [];
  }
  
  // If no filter is set, return all plans
  if (priceFilter.value.min === null && priceFilter.value.max === null) {
    return serverCatalog.value.plans;
  }
  
  return serverCatalog.value.plans.filter(plan => {
    const price = getNumericPrice(plan);
    
    // Skip plans without valid price
    if (price === null) {
      return false;
    }
    
    // Check min price filter
    if (priceFilter.value.min !== null && price < priceFilter.value.min) {
      return false;
    }
    
    // Check max price filter
    if (priceFilter.value.max !== null && price > priceFilter.value.max) {
      return false;
    }
    
    return true;
  });
});

// Computed property for total plans count
const totalPlansCount = computed(() => {
  return serverCatalog.value.plans ? serverCatalog.value.plans.length : 0;
});

// Computed property for filtered plans count
const filteredPlansCount = computed(() => {
  return filteredPlans.value.length;
});

// Function to trigger filtering (called on input change)
const filterServersByPrice = () => {
  // The filtering is handled by the computed property automatically
  console.log(`Filtering servers with price range: €${priceFilter.value.min || 0} - €${priceFilter.value.max || '∞'}`);
};

// Function to clear price filter
const clearPriceFilter = () => {
  priceFilter.value.min = null;
  priceFilter.value.max = null;
  console.log('Price filter cleared');
};

// Function to get property color
const getPropertyColor = (propName) => {
  const colorMap = {
    'cpu': 'blue',
    'ram': 'green',
    'disk': 'amber',
    'storage': 'amber',
    'bandwidth': 'purple',
    'network': 'purple',
    'location': 'red',
    'region': 'red',
    'datacenter': 'red'
  };
  
  const key = Object.keys(colorMap).find(key => propName.toLowerCase().includes(key));
  return key ? colorMap[key] : 'grey';
};

// Constants for cache management
const SERVERS_CACHE_KEY = 'ovhcart_servers_cache';
const CACHE_EXPIRY_MS = 1000 * 60 * 60; // 1 hour cache duration

// Function to check if cache is valid
const isCacheValid = (cachedData) => {
  if (!cachedData || !cachedData.timestamp) return false;
  
  const currentTime = new Date().getTime();
  const cacheTime = cachedData.timestamp;
  
  // Check if cache is still within expiry period
  return (currentTime - cacheTime) < CACHE_EXPIRY_MS;
};

// Function to get cached server data
const getCachedServers = (subsidiaryCode) => {
  try {
    const cacheString = localStorage.getItem(SERVERS_CACHE_KEY);
    if (!cacheString) return null;
    
    const cache = JSON.parse(cacheString);
    
    // If subsidiary doesn't match cached subsidiary, don't use cache
    if (cache.subsidiaryCode !== subsidiaryCode) return null;
    
    // Check if cache is still valid
    if (!isCacheValid(cache)) {
      console.log('Server cache expired');
      return null;
    }
    
    console.log('Using cached server data');
    return cache.data;
  } catch (err) {
    console.error('Error reading server cache:', err);
    return null;
  }
};

// Function to save server data to cache
const cacheServerData = (data, subsidiaryCode) => {
  try {
    const cacheData = {
      data,
      subsidiaryCode,
      timestamp: new Date().getTime()
    };
    
    localStorage.setItem(SERVERS_CACHE_KEY, JSON.stringify(cacheData));
    console.log('Servers data cached successfully');
  } catch (err) {
    console.error('Error caching server data:', err);
  }
};

// Fetch servers from OVH API with optional cache bypass
const fetchServers = async (forceRefresh = false) => {
  loading.value = true;
  serverError.value = null;
  
  // Get the current subsidiary code from the selected site
  const subsidiaryCode = props.selectedSite ? props.selectedSite.code : 'IE';
  
  // Try to get data from cache first (unless force refresh is requested)
  if (!forceRefresh) {
    const cachedData = getCachedServers(subsidiaryCode);
    if (cachedData) {
      serverCatalog.value = cachedData;
      lastRefreshTime.value = new Date();
      isFromCache.value = true;
      loading.value = false;
      return;
    }
  }
  
  // Mark that we're loading from API, not cache
  isFromCache.value = false;
  
  try {
    const headers = {
      // 'Authorization': `Bearer ${props['api-token']}`,
      'Content-Type': 'application/json'
    };
    
    const apiEndpoint = getApiEndpoint();
    const response = await fetch(`${apiEndpoint}/order/catalog/public/eco?ovhSubsidiary=${subsidiaryCode}`, {
      headers
    });
    
    if (!response.ok) {
      throw new Error(`Failed to fetch server catalog: ${response.status}`);
    }
    
    const data = await response.json();
    
    // Initialize addingToCart property for each plan
    if (data.plans && data.plans.length > 0) {
      data.plans.forEach(plan => {
        plan.addingToCart = false;
      });
    }
    
    // Cache the server data
    cacheServerData(data, subsidiaryCode);
    
    serverCatalog.value = data;
    lastRefreshTime.value = new Date();
    console.log('Server catalog for subsidiary', subsidiaryCode, ':', data);
  } catch (err) {
    console.error('Error fetching server catalog:', err);
    serverError.value = err;
    showError(err.message);
  } finally {
    loading.value = false;
  }
};

// Show success notification
const showSuccess = (message) => {
  // Can be implemented with Vuetify snackbar if needed
  console.log('Success:', message);
  emit('show-success', message);
};

// Show error notification
const showError = (message) => {
  // Can be implemented with Vuetify snackbar if needed
  console.error('Error:', message);
  emit('show-error', message);
};

// Function to add server to cart
const addServerToCart = async (plan) => {
  if (!props.activeCartId || !plan.planCode) {
    showError('Cannot add server to cart. Please select an active cart first.');
    return;
  }
  
  // Set loading state for this specific plan
  if (plan) {
    plan.addingToCart = true;
  }
  
  try {
    const headers = {
      'Authorization': `Bearer ${props.apiToken}`,
      'Content-Type': 'application/json'
    };
    
    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${props.activeCartId}/eco`;
    
    console.log('Adding server to cart:', plan.planCode, 'URL:', url);
    
    const payload = {
      duration: selectedDuration.value,
      planCode: plan.planCode,
      pricingMode: "default",
      quantity: 1
    };
    
    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(payload)
    });
    
    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to add server to cart: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }
    
    const result = await response.json();
    showSuccess(`Added ${plan.planCode} to cart successfully!`);
    console.log('Server added to cart result:', result);
    
    // Notify parent to refresh cart details
    emit('refresh-cart-details');
    
    // Close the details dialog if it's open
    if (showServerDetailsDialog.value && selectedServer.value && selectedServer.value.planCode === plan.planCode) {
      showServerDetailsDialog.value = false;
    }
    
    return result;
  } catch (err) {
    console.error('Error adding server to cart:', err);
    showError(`Error adding server to cart: ${err.message}`);
    return null;
  } finally {
    // Reset loading state
    if (plan) {
      plan.addingToCart = false;
    }
  }
};

// Function to add the currently selected server to cart (from details dialog)
const addSelectedServerToCart = () => {
  if (selectedServer.value) {
    addServerToCart(selectedServer.value);
  }
};

// Router for navigation
const router = useRouter();

// Function to navigate to server plan details page
const showServerDetails = (plan) => {
  if (plan && plan.planCode) {
    router.push(`/plan/${plan.planCode}`);
  }
};

// Function to add server by manually entered plan code
const addServerByPlanCode = async () => {
  if (!props.activeCartId || !manualPlanCode.value) {
    showError('Cannot add server to cart. Please select an active cart first and enter a plan code.');
    return;
  }
  
  addingManualPlan.value = true;
  
  try {
    const headers = {
      'Authorization': `Bearer ${props.apiToken}`,
      'Content-Type': 'application/json'
    };
    
    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${props.activeCartId}/eco`;
    
    console.log('Adding manual plan to cart:', manualPlanCode.value, 'URL:', url);
    
    const payload = {
      duration: manualDuration.value,
      planCode: manualPlanCode.value,
      pricingMode: "default",
      quantity: parseInt(manualQuantity.value, 10) || 1
    };
    
    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(payload)
    });
    
    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to add plan to cart: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }
    
    const result = await response.json();
    showSuccess(`Added ${manualPlanCode.value} to cart successfully!`);
    console.log('Manual plan added to cart result:', result);
    
    // Notify parent to refresh cart details
    emit('refresh-cart-details');
    
    // Reset and close dialog
    manualPlanCode.value = '';
    manualDuration.value = 'P1M';
    manualQuantity.value = 1;
    showManualPlanCodeDialog.value = false;
    
    return result;
  } catch (err) {
    console.error('Error adding manual plan to cart:', err);
    showError(`Error adding plan to cart: ${err.message}`);
    return null;
  } finally {
    addingManualPlan.value = false;
  }
};

// Watch for site changes to refresh servers
watch(() => props.selectedSite, (newSite, oldSite) => {
  if (newSite?.code !== oldSite?.code) {
    console.log('Site changed, refreshing server catalog...');
    fetchServers();
  }
}, { deep: true });

// Format the refresh time in a user-friendly way
const formatRefreshTime = (date) => {
  if (!date) return '';
  
  // Get current date for comparison
  const now = new Date();
  const refreshDate = new Date(date);
  
  // If same day, just show time
  if (now.toDateString() === refreshDate.toDateString()) {
    return refreshDate.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }
  
  // Otherwise show date and time
  return refreshDate.toLocaleString([], { 
    month: 'short', 
    day: 'numeric',
    hour: '2-digit', 
    minute: '2-digit'
  });
};

// Force refresh servers (ignoring cache)
const forceRefreshServers = () => {
  console.log('Forcing refresh of server data from API');
  fetchServers(true);
};

// Fetch servers on component mount if we have a token
onMounted(() => {
  fetchServers();
});
</script>

<style scoped>
.h-100 {
  height: 100%;
}
</style>
