<template>
  <v-container>
    <v-card>
      <v-card-title class="text-h4 font-weight-bold d-flex align-center">
        <v-icon icon="mdi-cart" color="primary" class="mr-2"></v-icon>
        OVH Cart
      </v-card-title>
      <v-tabs
        v-model="activeTab"
        bg-color="primary"
      >
        <v-tab
          v-for="tab in tabs"
          :key="tab.id"
          :value="tab.id"
        >
          <v-icon :icon="tab.icon" class="mr-2"></v-icon>
          {{ tab.name }}
        </v-tab>
      </v-tabs>

      <v-card-text>
        <v-window v-model="activeTab">
          <!-- Cart Tab -->
          <v-window-item value="cart">
            <v-card-title class="text-h5 py-4 d-flex align-center">
              Shopping Cart
              <v-spacer></v-spacer>
              <v-btn
                color="success"
                variant="elevated"
                prepend-icon="mdi-cart-plus"
                @click="createCart"
                :loading="creatingCart"
                :disabled="!apiToken"
              >
                Create New Cart
              </v-btn>
            </v-card-title>
            <v-card-text>
              <div v-if="loading">
                <v-progress-circular indeterminate color="primary"></v-progress-circular>
                <span class="ml-2">Loading cart data...</span>
              </div>
              <v-alert
                v-else-if="error"
                type="error"
                title="Error"
                :text="error.message || 'Failed to load cart data'"
              ></v-alert>
              <v-alert
                v-if="!activeCartId && !loading && cartItems.length > 0"
                type="warning"
                text="No active cart selected. Please select one from below."
                class="mb-4"
              ></v-alert>

              <!-- Active Cart Section -->
              <div v-if="activeCartId && activeCartDetails" class="mb-6">
                <v-card variant="outlined" class="mb-4">
                  <v-card-title>
                    <div class="d-flex align-center">
                      <span>Active Cart</span>
                      <v-chip
                        :color="activeCartDetails.readOnly ? 'error' : 'success'"
                        size="small"
                        class="ml-2"
                      >
                        {{ activeCartDetails.readOnly ? 'Read Only' : 'Active' }}
                      </v-chip>
                    </div>
                  </v-card-title>
                  <v-card-text>
                    <v-row>
                      <v-col cols="12" sm="6">
                        <v-list-item>
                          <template v-slot:prepend>
                            <v-icon icon="mdi-cart-outline" color="primary"></v-icon>
                          </template>
                          <v-list-item-title>Cart ID</v-list-item-title>
                          <v-list-item-subtitle>{{ activeCartDetails.cartId }}</v-list-item-subtitle>
                        </v-list-item>
                      </v-col>
                      <v-col cols="12" sm="6">
                        <v-list-item>
                          <template v-slot:prepend>
                            <v-icon icon="mdi-text-box-outline" color="primary"></v-icon>
                          </template>
                          <v-list-item-title>Description</v-list-item-title>
                          <v-list-item-subtitle>{{ activeCartDetails.description || 'No description' }}</v-list-item-subtitle>
                        </v-list-item>
                      </v-col>
                      <v-col cols="12" sm="6">
                        <v-list-item>
                          <template v-slot:prepend>
                            <v-icon icon="mdi-calendar-clock" color="primary"></v-icon>
                          </template>
                          <v-list-item-title>Expiration</v-list-item-title>
                          <v-list-item-subtitle>{{ formatDate(activeCartDetails.expire) }}</v-list-item-subtitle>
                        </v-list-item>
                      </v-col>
                      <v-col cols="12" sm="6">
                        <v-list-item>
                          <template v-slot:prepend>
                            <v-icon icon="mdi-package-variant-closed" color="primary"></v-icon>
                          </template>
                          <v-list-item-title>Items</v-list-item-title>
                          <v-list-item-subtitle>{{ activeCartDetails.items?.length || 0 }} items</v-list-item-subtitle>
                        </v-list-item>
                      </v-col>
                    </v-row>

                    <!-- Cart Items Section -->
                    <v-divider class="my-4"></v-divider>
                    <div v-if="activeCartDetails && activeCartDetails.items && activeCartDetails.items.length > 0">
                      <h3 class="text-subtitle-1 font-weight-bold mb-2">Cart Items</h3>
                      <v-card v-for="(itemId, index) in activeCartDetails.items" :key="index" class="mb-4" variant="outlined">
                        <v-card-item>
                          <v-card-title>
                            <div class="d-flex align-center">
                              <span>Item #{{ index + 1 }}</span>
                              <v-chip
                                v-if="cartItemDetails[itemId]"
                                :color="getItemStatusColor(cartItemDetails[itemId])"
                                size="small"
                                class="ml-2"
                              >
                                {{ cartItemDetails[itemId]?.readOnly ? 'Read Only' : 'Active' }}
                              </v-chip>
                            </div>
                          </v-card-title>

                          <v-card-text>
                            <div v-if="!cartItemDetails[itemId]" class="d-flex align-center">
                              <v-progress-circular indeterminate size="20" width="2" color="primary" class="mr-2"></v-progress-circular>
                              <span>Loading item details...</span>
                            </div>

                            <v-row v-else>
                              <v-col cols="12" sm="6">
                                <v-card-title>{{ cartItemDetails[itemId]?.description || cartItemDetails[itemId]?.planCode }}</v-card-title>
                                <v-card-subtitle v-if="cartItemDetails[itemId]?.description && cartItemDetails[itemId]?.planCode">{{ cartItemDetails[itemId]?.planCode }}</v-card-subtitle>

                                <v-chip
                                  v-if="cartItemDetails[itemId]?.settings?.quantity"
                                  color="primary"
                                  size="small"
                                  class="mt-2"
                                  prepend-icon="mdi-numeric"
                                >
                                  Quantity: {{ cartItemDetails[itemId]?.settings?.quantity }}
                                </v-chip>

                                <v-list density="compact" class="mt-2">
                                  <v-list-item>
                                    <template v-slot:prepend>
                                      <v-icon icon="mdi-card-bulleted-outline" color="primary"></v-icon>
                                    </template>
                                    <v-list-item-title>Product</v-list-item-title>
                                    <v-list-item-subtitle>{{ getProductName(cartItemDetails[itemId]?.productId) }}</v-list-item-subtitle>
                                  </v-list-item>
                                  <v-list-item>
                                    <template v-slot:prepend>
                                      <v-icon icon="mdi-tag" color="primary"></v-icon>
                                    </template>
                                    <v-list-item-title>Plan Code</v-list-item-title>
                                    <v-list-item-subtitle>{{ cartItemDetails[itemId]?.settings?.planCode }}</v-list-item-subtitle>
                                  </v-list-item>
                                </v-list>
                              </v-col>

                              <v-col cols="12" sm="6">
                                <v-list density="compact">
                                  <v-list-item v-if="cartItemDetails[itemId]?.prices">
                                    <template v-slot:prepend>
                                      <v-icon icon="mdi-currency-eur" color="primary"></v-icon>
                                    </template>
                                    <v-list-item-title>Price</v-list-item-title>
                                    <v-list-item-subtitle>{{ getItemPrice(cartItemDetails[itemId]) }}</v-list-item-subtitle>
                                  </v-list-item>

                                  <v-list-item>
                                    <template v-slot:prepend>
                                      <v-icon icon="mdi-cog-outline" color="primary"></v-icon>
                                    </template>
                                    <v-list-item-title>Settings</v-list-item-title>
                                    <v-list-item-subtitle>{{ cartItemDetails[itemId]?.configurations?.length || 0 }} configurations</v-list-item-subtitle>
                                  </v-list-item>
                                </v-list>
                              </v-col>
                            </v-row>

                            <!-- Pricing summary section if there are prices -->
                            <v-row>
                              <v-col v-if="cartItemDetails[itemId]?.prices && cartItemDetails[itemId]?.prices.length > 0" cols="12">
                                <v-card variant="outlined" class="mt-2">
                                  <v-card-title class="text-subtitle-1">
                                    <v-icon icon="mdi-currency-eur" color="primary" class="mr-2"></v-icon>
                                    Pricing Summary
                                  </v-card-title>
                                  <v-card-text>
                                    <v-table density="compact">
                                      <thead>
                                        <tr>
                                          <th>Type</th>
                                          <th>Price</th>
                                        </tr>
                                      </thead>
                                      <tbody>
                                        <tr v-for="(priceItem, priceIndex) in cartItemDetails[itemId].prices" :key="priceIndex">
                                          <td>{{ priceItem.label }}</td>
                                          <td>{{ priceItem.price.text || `${priceItem.price.value} ${priceItem.price.currencyCode}` }}</td>
                                        </tr>
                                      </tbody>
                                    </v-table>
                                  </v-card-text>
                                </v-card>
                              </v-col>

                              <!-- Datacenter Configuration -->
                              <v-col cols="12" class="mt-2">
                                <v-card variant="outlined">
                                  <v-card-title class="text-subtitle-1">
                                    <v-icon icon="mdi-map-marker" color="primary" class="mr-2"></v-icon>
                                    Datacenter Configuration
                                  </v-card-title>
                                  <v-card-text>
                                    <v-select
                                      v-model="selectedDatacenters[itemId]"
                                      :items="availableDatacenters"
                                      item-title="name"
                                      item-value="code"
                                      label="Select Datacenter"
                                      hint="Choose the datacenter location for this item"
                                      persistent-hint
                                      :disabled="activeCartDetails.readOnly || configuringDatacenter[itemId]"
                                      class="mb-2"
                                    >
                                      <template v-slot:item="{ item, props }">
                                        <v-list-item v-bind="props">
                                          <template v-slot:prepend>
                                            <v-icon :icon="getDatacenterIcon(item.raw.code)" color="primary"></v-icon>
                                          </template>
                                          <v-list-item-title>{{ item.raw.name }}</v-list-item-title>
                                          <v-list-item-subtitle>{{ item.raw.code }}</v-list-item-subtitle>
                                        </v-list-item>
                                      </template>
                                    </v-select>

                                    <p class="text-caption mt-1">Current datacenter:
                                      <v-chip size="x-small" color="info" v-if="getItemDatacenter(itemId)">
                                        {{ getItemDatacenter(itemId) }}
                                      </v-chip>
                                      <span v-else>Not configured</span>
                                    </p>

                                    <v-btn
                                      color="primary"
                                      variant="tonal"
                                      size="small"
                                      class="mt-2"
                                      :loading="configuringDatacenter[itemId]"
                                      :disabled="!selectedDatacenters[itemId] || activeCartDetails.readOnly"
                                      @click="configureItemDatacenter(itemId, selectedDatacenters[itemId])"
                                    >
                                      Apply Datacenter
                                    </v-btn>
                                  </v-card-text>
                                </v-card>
                              </v-col>

                              <!-- OS Configuration -->
                              <v-col cols="12" class="mt-2">
                                <v-card variant="outlined">
                                  <v-card-title class="text-subtitle-1">
                                    <v-icon icon="mdi-server-network" color="primary" class="mr-2"></v-icon>
                                    Operating System Configuration
                                  </v-card-title>
                                  <v-card-text>
                                    <v-select
                                      v-model="selectedOS[itemId]"
                                      :items="availableOS"
                                      item-title="name"
                                      item-value="code"
                                      label="Select Operating System"
                                      hint="Choose the OS for this server"
                                      persistent-hint
                                      :disabled="activeCartDetails.readOnly || configuringOS[itemId]"
                                      class="mb-2"
                                    >
                                      <template v-slot:item="{ item, props }">
                                        <v-list-item v-bind="props">
                                          <template v-slot:prepend>
                                            <v-icon :icon="getOSIcon(item.raw.code)" color="primary"></v-icon>
                                          </template>
                                          <v-list-item-title>{{ item.raw.name }}</v-list-item-title>
                                          <v-list-item-subtitle>{{ item.raw.code }}</v-list-item-subtitle>
                                        </v-list-item>
                                      </template>
                                    </v-select>

                                    <p class="text-caption mt-1">Current OS:
                                      <v-chip size="x-small" color="info" v-if="getItemOS(itemId)">
                                        {{ getItemOS(itemId) }}
                                      </v-chip>
                                      <span v-else>Not configured</span>
                                    </p>

                                    <v-btn
                                      color="primary"
                                      variant="tonal"
                                      size="small"
                                      class="mt-2"
                                      :loading="configuringOS[itemId]"
                                      :disabled="!selectedOS[itemId] || activeCartDetails.readOnly"
                                      @click="configureItemOS(itemId, selectedOS[itemId])"
                                    >
                                      Apply OS
                                    </v-btn>
                                  </v-card-text>
                                </v-card>
                              </v-col>

                              <!-- Configuration Details Section -->
                              <v-col cols="12">
                                <v-card variant="outlined" class="mt-2">
                                  <v-card-title class="text-subtitle-1 d-flex align-center">
                                    <v-icon icon="mdi-cog" color="primary" class="mr-2"></v-icon>
                                    Configuration Details
                                    <v-spacer></v-spacer>
                                    <v-btn
                                      v-if="!activeCartDetails.readOnly"
                                      icon="mdi-refresh"
                                      variant="text"
                                      size="small"
                                      color="primary"
                                      @click="fetchActiveCartDetails"
                                      :loading="loadingActiveCart"
                                      :disabled="activeCartDetails.readOnly"
                                    ></v-btn>
                                  </v-card-title>
                                  <v-card-text>
                                    <div v-if="!cartItemDetails[itemId]?.configurationDetails || cartItemDetails[itemId]?.configurationDetails.length === 0">
                                      <v-alert
                                        type="info"
                                        density="compact"
                                        variant="tonal"
                                        text="No configurations found for this item."
                                        class="mb-2"
                                      ></v-alert>
                                    </div>
                                    <v-table v-else density="compact">
                                      <thead>
                                        <tr>
                                          <th>Configuration ID</th>
                                          <th>Label</th>
                                          <th>Value</th>
                                          <th v-if="!activeCartDetails.readOnly">Actions</th>
                                        </tr>
                                      </thead>
                                      <tbody>
                                        <tr v-for="(config, configIndex) in cartItemDetails[itemId].configurationDetails" :key="configIndex">
                                          <td>{{ config.id }}</td>
                                          <td>{{ config.label }}</td>
                                          <td>{{ config.value }}</td>
                                          <td v-if="!activeCartDetails.readOnly">
                                            <div class="d-flex">
                                              <v-btn
                                                density="compact"
                                                icon="mdi-pencil"
                                                variant="text"
                                                color="primary"
                                                size="small"
                                                @click="openConfigEditor(itemId, config)"
                                                :disabled="editingConfiguration[itemId]"
                                                class="mr-1"
                                              ></v-btn>
                                              <v-btn
                                                density="compact"
                                                icon="mdi-delete"
                                                variant="text"
                                                color="error"
                                                size="small"
                                                @click="openDeleteConfirmation(itemId, config)"
                                                :disabled="editingConfiguration[itemId]"
                                              ></v-btn>
                                            </div>
                                          </td>
                                        </tr>
                                      </tbody>
                                    </v-table>

                                    <!-- Add new configuration section -->
                                    <div class="mt-4" v-if="!activeCartDetails.readOnly">
                                      <v-divider class="mb-2"></v-divider>
                                      <v-row>
                                        <v-col cols="12">
                                          <v-select
                                            v-model="newConfig.label"
                                            :items="commonConfigurations"
                                            label="Select Configuration Type"
                                            density="compact"
                                            class="mb-2"
                                            @update:model-value="handleConfigTypeChange"
                                          ></v-select>

                                          <v-alert
                                            v-if="configurationInfo[newConfig.label]"
                                            type="info"
                                            density="compact"
                                            variant="tonal"
                                            class="mb-2"
                                          >
                                            {{ configurationInfo[newConfig.label] }}
                                          </v-alert>
                                        </v-col>
                                        <v-col cols="12">
                                          <div class="d-flex align-center">
                                            <v-text-field
                                              v-if="!isInCommonConfigurations(newConfig.label)"
                                              v-model="newConfig.label"
                                              label="Custom Configuration Name"
                                              density="compact"
                                              class="mr-2"
                                              hide-details
                                            ></v-text-field>
                                            <v-text-field
                                              v-model="newConfig.value"
                                              label="Configuration Value"
                                              density="compact"
                                              class="mr-2"
                                              hide-details
                                            ></v-text-field>
                                            <v-btn
                                              color="success"
                                              size="small"
                                              density="compact"
                                              @click="addItemConfiguration(itemId, newConfig.label, newConfig.value)"
                                              :loading="addingConfiguration[itemId]"
                                              :disabled="!newConfig.label || !newConfig.value"
                                            >
                                              Add
                                            </v-btn>
                                          </div>
                                        </v-col>
                                      </v-row>
                                    </div>
                                  </v-card-text>
                                </v-card>
                              </v-col>
                            </v-row>

                            <!-- Action buttons -->
                            <v-card-actions v-if="cartItemDetails[itemId]">
                              <v-btn
                                color="error"
                                variant="text"
                                prepend-icon="mdi-delete"
                                @click="removeItemFromCart(itemId)"
                                :loading="deletingItems[itemId]"
                                :disabled="activeCartDetails.readOnly"
                              >
                                Remove
                              </v-btn>
                            </v-card-actions>
                          </v-card-text>
                        </v-card-item>
                      </v-card>
                    </div>
                    <div v-else>
                      <v-alert type="info" text="No items in this cart" class="mt-2"></v-alert>
                    </div>
                  </v-card-text>
                  <v-card-actions>
                    <v-btn
                      color="primary"
                      variant="text"
                      prepend-icon="mdi-refresh"
                      @click="fetchActiveCartDetails"
                      :loading="loadingActiveCart"
                    >
                      Refresh Cart Details
                    </v-btn>
                    <v-spacer></v-spacer>
                    <v-btn
                      color="success"
                      variant="elevated"
                      prepend-icon="mdi-cash-register"
                      @click="checkoutCart"
                      :loading="checkingOut"
                      :disabled="activeCartDetails.readOnly || !activeCartDetails.items || activeCartDetails.items.length === 0"
                    >
                      Checkout
                    </v-btn>
                    <v-btn
                      color="error"
                      variant="text"
                      prepend-icon="mdi-close-circle"
                      @click="clearActiveCart"
                      :disabled="activeCartDetails.readOnly"
                    >
                      Clear Active Cart
                    </v-btn>
                  </v-card-actions>
                </v-card>
              </div>

              <!-- Available Carts -->
              <h3 class="text-subtitle-1 font-weight-bold mb-2">Available Carts</h3>
              <v-alert
                v-if="cartItems.length === 0 && !loading"
                type="info"
                text="Your cart is empty"
                class="mb-4"
              ></v-alert>
              <v-list v-else lines="three">
                <v-list-item
                  v-for="(item, index) in cartItems"
                  :key="index"
                  :title="item.description || 'Cart Item'"
                  :subtitle="'ID: ' + item.cartId"
                  :active="activeCartId === item.cartId"
                  @click="setActiveCart(item.cartId)"
                >
                  <template v-slot:prepend>
                    <v-chip
                      :color="item.readOnly ? 'error' : 'success'"
                      size="small"
                      class="text-caption"
                    >
                      {{ item.readOnly ? 'Read Only' : 'Active' }}
                    </v-chip>
                    <v-icon
                      v-if="activeCartId === item.cartId"
                      icon="mdi-check-circle"
                      color="primary"
                      class="ml-2"
                    ></v-icon>
                  </template>
                  <template v-slot:append>
                    <div class="d-flex align-center">
                      <span class="text-caption text-grey me-2">
                        Expires: {{ formatDate(item.expire) }}
                      </span>
                      <v-btn
                        icon="mdi-delete"
                        variant="text"
                        color="error"
                        density="compact"
                        @click="removeFromCart(item)"
                        :disabled="item.readOnly"
                      ></v-btn>
                    </div>
                  </template>
                </v-list-item>
              </v-list>
            </v-card-text>
            <v-card-actions>
              <v-btn
                color="primary"
                variant="elevated"
                prepend-icon="mdi-refresh"
                @click="fetchCart"
                :loading="loading"
                :disabled="!apiToken"
              >
                Refresh Cart
              </v-btn>
            </v-card-actions>
          </v-window-item>

          <!-- Servers Tab -->
          <v-window-item value="servers">
            <servers
              :api-token="apiToken"
              :selected-site="selectedSite"
              :active-cart-id="activeCartId"
              @refresh-cart-details="fetchActiveCartDetails"
              @show-success="showSuccess"
              @show-error="showError"
            />
          </v-window-item>

          <!-- User Profile Tab -->
          <v-window-item value="me">
            <me
              :api-token="apiToken"
            />
          </v-window-item>

          <!-- Settings Tab -->
          <v-window-item value="settings">
            <Settings
              v-model:api-token="apiToken"
              v-model:selected-site="selectedSite"
              @settings-saved="fetchCart"
            />
          </v-window-item>

          <!-- About Tab -->
          <v-window-item value="about">
            <v-card-title class="text-h5 py-4">About OVH Cart</v-card-title>
            <v-card-text>
              <v-card variant="outlined" class="mb-4">
                <v-card-text>
                  <div class="d-flex align-center mb-4">
                    <v-icon icon="mdi-github" color="primary" size="large" class="mr-2"></v-icon>
                    <h3 class="text-h6">GitHub Repository</h3>
                  </div>
                  <p class="mb-2">OVH Cart is an open-source project hosted on GitHub.</p>
                  <v-btn
                    color="primary"
                    variant="outlined"
                    prepend-icon="mdi-github"
                    href="https://github.com/orvice/ovhcart"
                    target="_blank"
                  >
                    View on GitHub
                  </v-btn>
                </v-card-text>
              </v-card>
            </v-card-text>
          </v-window-item>
        </v-window>
      </v-card-text>
    </v-card>

    <!-- Add the new components here -->
    <v-dialog
      v-model="showCreateCartDialog"
      max-width="600"
    >
      <v-card>
        <v-card-title class="text-h5">Create New Cart</v-card-title>
        <v-card-text>
          <v-form @submit.prevent="submitCreateCart">
            <v-text-field
              v-model="newCart.description"
              label="Description"
              hint="Enter a description for your new cart"
              persistent-hint
              :rules="[v => !!v || 'Description is required']"
              required
              class="mb-4"
            ></v-text-field>

            <v-select
              v-model="newCart.ovhSubsidiary"
              label="OVH Subsidiary"
              :items="subsidiaries"
              hint="Select OVH subsidiary"
              persistent-hint
              required
              class="mb-4"
            ></v-select>

            <v-text-field
              v-model="newCart.expire"
              label="Expiration"
              type="datetime-local"
              hint="Cart will expire at this date/time (format: yyyy-MM-ddThh:mm:ss)"
              persistent-hint
              required
              class="mb-4"
            ></v-text-field>

            <v-switch
              v-model="newCart.readOnly"
              label="Read Only"
              hint="Make this cart read-only"
              persistent-hint
            ></v-switch>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="error"
            variant="text"
            @click="showCreateCartDialog = false"
          >
            Cancel
          </v-btn>
          <v-btn
            color="success"
            variant="elevated"
            @click="submitCreateCart"
            :loading="creatingCart"
          >
            Create
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog
      v-model="showManualPlanCodeDialog"
      max-width="500px"
    >
      <v-card>
        <v-card-title class="text-h5">Add Server by Plan Code</v-card-title>
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
            ></v-text-field>

            <v-select
              v-model="manualDuration"
              :items="durations"
              item-title="text"
              item-value="value"
              label="Duration"
              hint="Select contract duration"
              persistent-hint
              class="mb-4"
            ></v-select>

            <v-text-field
              v-model="manualQuantity"
              label="Quantity"
              type="number"
              min="1"
              hint="Number of items to add"
              persistent-hint
              :rules="[v => !!v && v > 0 || 'Quantity must be at least 1']"
              required
            ></v-text-field>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
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

    <v-dialog
      v-model="showServerDetailsDialog"
      max-width="600"
    >
      <v-card>
        <v-card-title>Server Details</v-card-title>
        <v-card-text>
          <!-- Server details content -->
        </v-card-text>
      </v-card>
    </v-dialog>

    <v-dialog
      v-model="confirmDeleteDialog"
      max-width="400"
    >
      <v-card>
        <v-card-title>Confirm Deletion</v-card-title>
        <v-card-text>
          Are you sure you want to delete this configuration?
        </v-card-text>
        <v-card-actions>
          <v-btn
            color="error"
            @click="confirmDelete"
          >
            Delete
          </v-btn>
          <v-btn
            @click="confirmDeleteDialog = false"
          >
            Cancel
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-snackbar
      v-model="showSuccessSnackbar"
      :timeout="3000"
      color="success"
    >
      {{ successMessage }}
    </v-snackbar>

    <v-snackbar
      v-model="showErrorSnackbar"
      :timeout="3000"
      color="error"
    >
      {{ errorMessage }}
    </v-snackbar>

    <!-- After all other dialogs, add the checkout dialog -->
    <v-dialog
      v-model="showCheckoutDialog"
      max-width="500px"
    >
      <v-card>
        <v-card-title>
          <v-icon icon="mdi-check-circle" color="success" class="mr-2"></v-icon>
          Checkout Successful
        </v-card-title>
        <v-card-text>
          <div class="text-body-1 mb-4">
            Your cart has been successfully checked out. Please use the link below to complete your payment.
          </div>

          <v-alert
            type="info"
            variant="tonal"
            icon="mdi-information"
            class="mb-4"
          >
            After payment, your order will be processed and you will receive confirmation details.
          </v-alert>

          <v-card variant="outlined" class="pa-3 mb-4">
            <v-card-title class="text-subtitle-1 d-flex align-center">
              <v-icon icon="mdi-link" color="primary" class="mr-2"></v-icon>
              Payment URL
            </v-card-title>
            <v-card-text>
              <div class="d-flex align-center">
                <v-text-field
                  v-model="checkoutUrl"
                  readonly
                  variant="outlined"
                  class="flex-grow-1 mr-2"
                  dense
                ></v-text-field>
                <v-btn
                  icon="mdi-content-copy"
                  size="small"
                  @click="copyToClipboard(checkoutUrl)"
                  title="Copy to clipboard"
                ></v-btn>
              </div>
            </v-card-text>
          </v-card>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="primary"
            variant="text"
            @click="showCheckoutDialog = false"
          >
            Close
          </v-btn>
          <v-btn
            color="success"
            variant="elevated"
            prepend-icon="mdi-open-in-new"
            :href="checkoutUrl"
            target="_blank"
          >
            Open Payment Page
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import Me from './Me.vue'
import Servers from './Servers.vue'
import Settings from './Settings.vue'
import { availableSites } from '../config/sites'

const tabs = [
  { id: 'cart', name: 'Shopping Cart', icon: 'mdi-cart' },
  { id: 'servers', name: 'Servers', icon: 'mdi-server' },
  { id: 'me', name: 'User Profile', icon: 'mdi-account' },
  { id: 'settings', name: 'Settings', icon: 'mdi-cog' },
  { id: 'about', name: 'About', icon: 'mdi-information' }
]

const activeTab = ref('cart')
const cartItems = ref([])
const apiToken = ref('')
const loading = ref(false)
const error = ref(null)

// Active cart tracking
const activeCartId = ref(null)
const activeCartDetails = ref(null)
const loadingActiveCart = ref(false)

// Cart item details
const cartItemDetails = ref({});
const deletingItems = ref({});

// 用于跟踪项目详情加载状态
const loadingItemDetails = ref(false);

// 监听购物车ID变化，当ID变化时重置和获取购物车详情
watch(activeCartId, (newCartId) => {
  console.log('Active cart ID changed to:', newCartId);
  // 清空之前的购物车项目详情
  cartItemDetails.value = {};

  if (newCartId) {
    // 当有新的购物车ID时，获取详情
    fetchActiveCartDetails();
  } else {
    // 当购物车ID被清空时，重置相关状态
    activeCartDetails.value = null;
  }
});

// Create cart related refs
const showCreateCartDialog = ref(false)
const creatingCart = ref(false)
const newCart = ref({
  description: '',
  ovhSubsidiary: 'IE',
  readOnly: false,
  expire: getDefaultExpireDate()
})

// Date picker related refs
const showExpiryPicker = ref(false)
const showExpiryTimePicker = ref(false)
const expireDatePart = ref('')
const expireTimePart = ref('')

// Subsidiary options
const subsidiaries = ['EU', 'CA', 'US', 'IE']



// Snackbar notifications
const showSuccessSnackbar = ref(false)
const showErrorSnackbar = ref(false)
const successMessage = ref('')
const errorMessage = ref('')

// Server catalog data
const serverCatalog = ref(null)
const loadingServers = ref(false)
const serverError = ref(null)
const selectedServer = ref(null)
const showServerDetailsDialog = ref(false)

// Site selection
const selectedSite = ref(null)

// Theme selection
const selectedTheme = ref('auto')

// In the script section, add these variables
const selectedDuration = ref('P1M'); // Default to 1 month
const durations = [
  { value: 'P1M', text: '1 Month' },
  { value: 'P3M', text: '3 Months' },
  { value: 'P6M', text: '6 Months' },
  { value: 'P12M', text: '12 Months' }
];

// Datacenter configuration
const selectedDatacenters = ref({});
const configuringDatacenter = ref({});
const availableDatacenters = [
  { code: 'bhs', name: 'Beauharnois, Canada' },
  { code: 'rbx', name: 'Roubaix, France' },
  { code: 'gra', name: 'Gravelines, France' },
  { code: 'fra', name: 'Frankfurt, Germany' }
];

// OS configuration
const selectedOS = ref({});
const configuringOS = ref({});
const availableOS = [
  { code: 'none_64.en', name: 'No OS (64-bit)' }
  // More OS options can be added here when available
];

// Configuration editing
const editingConfiguration = ref({});
const addingConfiguration = ref({});
const deleteConfigLoading = ref({});
const newConfig = ref({ label: '', value: '' });
const configEditorDialog = ref(false);
const currentEditConfig = ref({ itemId: null, config: null, newValue: '', configId: null });

// Common configurations
const commonConfigurations = [
  'dedicated_datacenter',
  'hostname',
  'storage_size',
  'cpu_count',
  'memory_size',
  'os_type',
  'network_type',
  'backup_option',
  'monitoring_level',
  'custom_configuration' // This will allow users to enter a custom configuration name
];

// Configuration info descriptions
const configurationInfo = {
  'dedicated_datacenter': 'Set the datacenter where your server will be located (bhs, rbx, gra, fra).',
  'hostname': 'Set the hostname for your server, e.g., server1.example.com',
  'storage_size': 'Specify storage size in GB, e.g., 500',
  'cpu_count': 'Specify the number of CPU cores, e.g., 4',
  'memory_size': 'Specify RAM size in GB, e.g., 32',
  'os_type': 'Specify operating system, e.g., ubuntu20, centos8, windows2019',
  'network_type': 'Specify network configuration, e.g., public, private, hybrid',
  'backup_option': 'Specify backup option, e.g., daily, weekly, none',
  'monitoring_level': 'Specify monitoring level, e.g., basic, standard, premium'
};

// Available configurations
const loadingAvailableConfigurations = ref({});
const itemAvailableConfigurations = ref({});

// Add confirmation dialog refs
const confirmDeleteDialog = ref(false);
const configToDelete = ref({ itemId: null, configId: null, label: '' });

// On mounted, load saved site from localStorage
onMounted(() => {
  // Set default site first to avoid undefined errors
  selectedSite.value = availableSites.find(site => site.code === 'IE')

  const savedToken = localStorage.getItem('ovhApiToken')
  if (savedToken) {
    apiToken.value = savedToken

    // Load active cart ID
    const savedCartId = localStorage.getItem('ovhActiveCartId')
    if (savedCartId) {
      activeCartId.value = savedCartId
      fetchActiveCartDetails()
    }

    // Load saved site
    const savedSiteCode = localStorage.getItem('ovhSiteCode')
    if (savedSiteCode) {
      const foundSite = availableSites.find(site => site.code === savedSiteCode)
      if (foundSite) {
        selectedSite.value = foundSite
      }
    }

    // Load saved theme
    const savedTheme = localStorage.getItem('ovhTheme')
    if (savedTheme) {
      selectedTheme.value = savedTheme
      applyTheme(savedTheme)
    } else {
      // Default to auto theme
      applyTheme('auto')
    }

    fetchCart()
  }
})

// Function to format date
const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  return date.toLocaleString()
}

// Function to show success notification
const showSuccess = (message) => {
  successMessage.value = message
  showSuccessSnackbar.value = true
}

// Function to show error notification
const showError = (message) => {
  errorMessage.value = message
  showErrorSnackbar.value = true
}

// Function to get API endpoint based on selected site
const getApiEndpoint = () => {
  // Always use eu.api.ovh.com regardless of site selection
  return 'https://eu.api.ovh.com/v1';
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

// Function to fetch cart data
const fetchCart = async () => {
  if (!apiToken.value) return

  loading.value = true
  error.value = null

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    }

    // First, get the cart list
    const apiEndpoint = getApiEndpoint()
    const cartListResponse = await fetch(`${apiEndpoint}/order/cart`, {
      headers
    })

    if (!cartListResponse.ok) {
      throw new Error(`Failed to fetch cart list: ${cartListResponse.status}`)
    }

    const cartIds = await cartListResponse.json()

    // If there are carts, fetch details for each
    if (cartIds && cartIds.length) {
      const cartDetails = []

      for (const cartId of cartIds) {
        const cartDetailResponse = await fetch(`${apiEndpoint}/order/cart/${cartId}`, {
          headers
        })

        if (cartDetailResponse.ok) {
          const detail = await cartDetailResponse.json()
          cartDetails.push(detail)
        }
      }

      cartItems.value = cartDetails
    } else {
      cartItems.value = []
    }
  } catch (err) {
    console.error('Error fetching cart:', err)
    error.value = err
    showError(`Error fetching cart: ${err.message}`)
  } finally {
    loading.value = false
  }
}

const removeFromCart = (item) => {
  // Implement cart item removal functionality
  console.log('Remove item from cart:', item)
}

// Function to get default expire date (current time + 7 days) in proper format for datetime-local input
function getDefaultExpireDate() {
  const date = new Date();
  date.setDate(date.getDate() + 7);

  // Format as yyyy-MM-ddThh:mm:ss (compatible with datetime-local input)
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  const hours = String(date.getHours()).padStart(2, '0');
  const minutes = String(date.getMinutes()).padStart(2, '0');
  const seconds = String(date.getSeconds()).padStart(2, '0');

  return `${year}-${month}-${day}T${hours}:${minutes}:${seconds}`;
}


// Function to show create cart dialog
const createCart = () => {
  newCart.value = {
    description: '',
    ovhSubsidiary: selectedSite.value ? selectedSite.value.code : 'IE',
    readOnly: false,
    expire: getDefaultExpireDate() // This now returns properly formatted date string
  }

  showCreateCartDialog.value = true
}

// Function to submit new cart creation
const submitCreateCart = async () => {
  if (!apiToken.value) return;

  creatingCart.value = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    // Convert the datetime-local input value to ISO 8601 format for the API
    let expireDate = new Date(newCart.value.expire);
    let expireISOString = expireDate.toISOString();

    const apiEndpoint = getApiEndpoint();
    const response = await fetch(`${apiEndpoint}/order/cart`, {
      method: 'POST',
      headers,
      body: JSON.stringify({
        ovhSubsidiary: newCart.value.ovhSubsidiary,
        description: newCart.value.description,
        readOnly: newCart.value.readOnly,
        expire: expireISOString // Use ISO string format for the API
      })
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to create cart: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const result = await response.json();

    // Store the cart ID in localStorage and set as active
    localStorage.setItem('ovhActiveCartId', result.cartId);
    activeCartId.value = result.cartId;
    activeCartDetails.value = result;

    showSuccess(`Cart created successfully! Cart ID: ${result.cartId}`);
    showCreateCartDialog.value = false;

    // Refresh cart data to include the new cart
    await fetchCart();
  } catch (err) {
    console.error('Error creating cart:', err);
    showError(`Error creating cart: ${err.message}`);
  } finally {
    creatingCart.value = false;
  }
}

// Function to set active cart
const setActiveCart = (cartId) => {
  activeCartId.value = cartId
  localStorage.setItem('ovhActiveCartId', cartId)
  fetchActiveCartDetails()
  showSuccess(`Cart ${cartId} set as active`)
}

// Function to clear active cart
const clearActiveCart = () => {
  activeCartId.value = null
  activeCartDetails.value = null
  localStorage.removeItem('ovhActiveCartId')
  showSuccess('Active cart cleared')
}

// Function to fetch active cart details
const fetchActiveCartDetails = async () => {
  if (!apiToken.value || !activeCartId.value) return

  loadingActiveCart.value = true

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    }

    const apiEndpoint = getApiEndpoint()
    const response = await fetch(`${apiEndpoint}/order/cart/${activeCartId.value}`, {
      headers
    })

    if (!response.ok) {
      throw new Error(`Failed to fetch cart details: ${response.status}`)
    }

    const cartDetails = await response.json()
    activeCartDetails.value = cartDetails
    console.log('Active cart details:', cartDetails)

    // Fetch details for each item
    if (cartDetails.items && cartDetails.items.length > 0) {
      fetchCartItemDetails(cartDetails.items);
    }
  } catch (err) {
    console.error('Error fetching active cart details:', err)
    showError(`Error fetching active cart details: ${err.message}`)
    activeCartDetails.value = null
  } finally {
    loadingActiveCart.value = false
  }
}

// Function to fetch item details
const fetchCartItemDetails = async (itemIds) => {
  if (!apiToken.value || !activeCartId.value || !itemIds || itemIds.length === 0) return;

  loadingItemDetails.value = true;
  console.log('Fetching details for cart items:', itemIds);

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();

    // 初始化临时存储对象，确保Vue的响应式系统能够正确检测到所有变更
    const tempDetails = {};

    // Fetch details for each item in parallel
    const promises = itemIds.map(async (itemId) => {
      try {
        console.log(`Fetching details for item ${itemId}...`);
        const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}`;
        const response = await fetch(url, { headers });

        if (!response.ok) {
          console.error(`Failed to fetch item ${itemId} details:`, response.status);
          tempDetails[itemId] = { error: `Failed to load: ${response.status}` };
          return;
        }

        const itemDetails = await response.json();
        console.log(`Received item ${itemId} details:`, itemDetails);

        // 存储到临时对象
        tempDetails[itemId] = itemDetails;

        // Fetch configuration details if configurations array exists
        if (itemDetails.configurations && Array.isArray(itemDetails.configurations) && itemDetails.configurations.length > 0) {
          // Store the configuration IDs
          const configIds = [...itemDetails.configurations];
          // Initialize configurations array to store detailed config objects
          tempDetails[itemId].configurationDetails = [];

          // Fetch each configuration detail
          const configPromises = configIds.map(async (configId) => {
            try {
              const configUrl = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration/${configId}`;
              const configResponse = await fetch(configUrl, { headers });

              if (!configResponse.ok) {
                console.error(`Failed to fetch configuration ${configId} details:`, configResponse.status);
                return;
              }

              const configDetails = await configResponse.json();
              console.log(`Configuration ${configId} details:`, configDetails);

              // Add to the item's configuration details array
              tempDetails[itemId].configurationDetails.push(configDetails);
            } catch (err) {
              console.error(`Error fetching configuration ${configId} details:`, err);
            }
          });

          await Promise.all(configPromises);
        }
      } catch (err) {
        console.error(`Error fetching item ${itemId} details:`, err);
        tempDetails[itemId] = { error: err.message };
      }
    });

    // 等待所有Promise完成
    await Promise.all(promises);

    // 一次性将所有结果更新到响应式对象，确保Vue能正确响应变化
    cartItemDetails.value = { ...tempDetails };
    console.log('All cart item details loaded:', cartItemDetails.value);
  } catch (err) {
    console.error('Error in fetchCartItemDetails:', err);
    showError(`Error loading cart items: ${err.message}`);
  } finally {
    loadingItemDetails.value = false;
  }
}

// Function to remove item from cart
const removeItemFromCart = async (itemId) => {
  if (!apiToken.value || !activeCartId.value) return;

  deletingItems.value[itemId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}`;

    const response = await fetch(url, {
      method: 'DELETE',
      headers
    });

    if (!response.ok) {
      throw new Error(`Failed to remove item: ${response.status}`);
    }

    showSuccess('Item removed from cart successfully');

    // Remove the item from cartItemDetails
    delete cartItemDetails.value[itemId];

    // Refresh cart details
    await fetchActiveCartDetails();
  } catch (err) {
    console.error('Error removing item from cart:', err);
    showError(`Error removing item from cart: ${err.message}`);
  } finally {
    deletingItems.value[itemId] = false;
  }
}

// Function to format duration string
const formatDuration = (duration) => {
  if (!duration) return 'N/A';

  // Convert ISO 8601 duration format to human-readable
  const regex = /P(?:(\d+)Y)?(?:(\d+)M)?(?:(\d+)D)?(?:T(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?)?/;
  const matches = duration.match(regex);

  if (!matches) return duration;

  const [, years, months, days, hours, minutes, seconds] = matches;
  const parts = [];

  if (years) parts.push(`${years} Year${years > 1 ? 's' : ''}`);
  if (months) parts.push(`${months} Month${months > 1 ? 's' : ''}`);
  if (days) parts.push(`${days} Day${days > 1 ? 's' : ''}`);
  if (hours) parts.push(`${hours} Hour${hours > 1 ? 's' : ''}`);
  if (minutes) parts.push(`${minutes} Minute${minutes > 1 ? 's' : ''}`);
  if (seconds) parts.push(`${seconds} Second${seconds > 1 ? 's' : ''}`);

  return parts.join(', ') || duration;
};

// Function to get item status color
const getItemStatusColor = (item) => {
  if (item?.readOnly) {
    return 'error';
  }
  return 'success';
};

// Function to get item price
const getItemPrice = (item) => {
  if (!item?.prices || item.prices.length === 0) {
    return 'Price not available';
  }

  // Try to find the total price first
  const totalPrice = item.prices.find(p => p.label === 'TOTAL');
  if (totalPrice && totalPrice.price) {
    return totalPrice.price.text || `${totalPrice.price.value} ${totalPrice.price.currencyCode}`;
  }

  // Fall back to the first price
  const firstPrice = item.prices[0];
  if (firstPrice && firstPrice.price) {
    return firstPrice.price.text || `${firstPrice.price.value} ${firstPrice.price.currencyCode}`;
  }

  return 'Price not available';
};

// Function to fetch server catalog
const fetchServers = async () => {
  if (!apiToken.value) return

  loadingServers.value = true
  serverError.value = null

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    }

    // Get the current subsidiary code from the selected site
    const subsidiaryCode = selectedSite.value ? selectedSite.value.code : 'IE'

    const apiEndpoint = getApiEndpoint()
    const response = await fetch(`${apiEndpoint}/order/catalog/public/eco?ovhSubsidiary=${subsidiaryCode}`, {
      headers
    })

    if (!response.ok) {
      throw new Error(`Failed to fetch server catalog: ${response.status}`)
    }

    const data = await response.json()

    // Initialize addingToCart property for each plan
    if (data.plans && data.plans.length > 0) {
      data.plans.forEach(plan => {
        plan.addingToCart = false;
      });
    }

    serverCatalog.value = data
    console.log('Server catalog for subsidiary', subsidiaryCode, ':', data)
  } catch (err) {
    console.error('Error fetching server catalog:', err)
    serverError.value = err
    showError(`Error fetching server catalog: ${err.message}`)
  } finally {
    loadingServers.value = false
  }
}

// Watch for site changes to refresh servers if on servers tab
watch(selectedSite, (newSite, oldSite) => {
  if (newSite?.code !== oldSite?.code && activeTab.value === 'servers') {
    console.log('Site changed, refreshing server catalog...');
    fetchServers();
  }
});

// Function to get plan price
const getPlanPrice = (plan) => {
  if (!plan.pricings || plan.pricings.length === 0) {
    return 'Price not available'
  }

  const pricing = plan.pricings[0]
  if (pricing.price) {
    return `${pricing.price.text || `${pricing.price.value} ${pricing.price.currencyCode}`}`
  }

  return 'Price not available'
}

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
  }

  const key = Object.keys(colorMap).find(key => propName.toLowerCase().includes(key))
  return key ? colorMap[key] : 'grey'
}

// Function to add server to cart
const addServerToCart = async (plan) => {
  if (!activeCartId.value || !plan.planCode) {
    showError('Cannot add server to cart. Please select an active cart first.')
    return
  }

  // Set loading state for this specific plan
  if (plan) {
    plan.addingToCart = true;
  }

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    }

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/eco`;

    console.log('Adding server to cart:', plan.planCode, 'URL:', url);

    const payload = {
      duration: selectedDuration.value, // Use selected duration
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

    // Refresh cart details
    await fetchActiveCartDetails();

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
}

// Function to show server details
const showServerDetails = (plan) => {
  selectedServer.value = plan;
  selectedDuration.value = 'P1M'; // Reset to default 1 month
  showServerDetailsDialog.value = true;
}

// Fetch servers when token is available and tab is changed to servers
watch(activeTab, (newTab) => {
  if (newTab === 'servers' && apiToken.value && (!serverCatalog.value || serverCatalog.value.plans?.length === 0)) {
    fetchServers()
  }
});

// Function to get a readable product name from product ID
const getProductName = (productId) => {
  if (!productId) return 'N/A';

  const productMap = {
    'eco': 'Eco Server',
    'domain': 'Domain Name',
    'dedicated': 'Dedicated Server',
    'vps': 'Virtual Private Server',
    'cloud': 'Cloud Service',
    'hosting': 'Web Hosting'
  };

  return productMap[productId] || productId;
};

// Function to get datacenter icon based on code
const getDatacenterIcon = (code) => {
  const iconMap = {
    'bhs': 'mdi-map-marker-radius',
    'rbx': 'mdi-map-marker-radius',
    'gra': 'mdi-map-marker-radius',
    'fra': 'mdi-map-marker-radius'
  };

  return iconMap[code] || 'mdi-map-marker';
};

// Function to get the current datacenter for an item
const getItemDatacenter = (itemId) => {
  if (!cartItemDetails.value[itemId] || !cartItemDetails.value[itemId].configurations) {
    return null;
  }

  const datacenterConfig = cartItemDetails.value[itemId].configurations.find(
    config => config.label === 'dedicated_datacenter'
  );

  return datacenterConfig ? datacenterConfig.value : null;
};

// Function to configure datacenter for a cart item
const configureItemDatacenter = async (itemId, datacenterCode) => {
  if (!apiToken.value || !activeCartId.value || !itemId || !datacenterCode) {
    showError('Cannot configure datacenter. Missing required information.');
    return;
  }

  // Set loading state
  configuringDatacenter.value[itemId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration`;

    const payload = {
      label: "dedicated_datacenter",
      value: datacenterCode
    };

    console.log(`Configuring datacenter for item ${itemId} to ${datacenterCode}:`, payload);

    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(payload)
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to configure datacenter: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const result = await response.json();
    showSuccess(`Datacenter configured to ${datacenterCode} successfully!`);
    console.log('Datacenter configuration result:', result);

    // Refresh cart item details
    await fetchActiveCartDetails();

    return result;
  } catch (err) {
    console.error('Error configuring datacenter:', err);
    showError(`Error configuring datacenter: ${err.message}`);
    return null;
  } finally {
    // Reset loading state
    configuringDatacenter.value[itemId] = false;
  }
};

// Function to open the configuration editor dialog
const openConfigEditor = (itemId, config) => {
  currentEditConfig.value = {
    itemId: itemId,
    config: { ...config },
    newValue: config.value,
    configId: config.id
  };
  configEditorDialog.value = true;
};

// Function to update an item configuration
const updateItemConfiguration = async (itemId, label, value, configId = null) => {
  if (!apiToken.value || !activeCartId.value || !itemId) {
    showError('Cannot update configuration. Missing required information.');
    return;
  }

  // Set loading state
  editingConfiguration.value[itemId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    let url;
    let method;

    // If configId is provided, update existing configuration
    if (configId) {
      url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration/${configId}`;
      method = 'PUT';
    } else {
      // Add new configuration
      url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration`;
      method = 'POST';
    }

    const payload = {
      label: label,
      value: value
    };

    console.log(`${configId ? 'Updating' : 'Adding'} configuration for item ${itemId}:`, payload);

    const response = await fetch(url, {
      method: method,
      headers,
      body: JSON.stringify(payload)
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to ${configId ? 'update' : 'add'} configuration: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const result = await response.json();
    showSuccess(`Configuration ${label} ${configId ? 'updated' : 'added'} successfully!`);
    console.log('Configuration update result:', result);

    // Refresh cart item details
    await fetchActiveCartDetails();

    // Reset the new config form
    if (label === newConfig.value.label) {
      newConfig.value = { label: '', value: '' };
    }

    // Close dialog if it was from the dialog
    configEditorDialog.value = false;

    return result;
  } catch (err) {
    console.error(`Error ${configId ? 'updating' : 'adding'} configuration:`, err);
    showError(`Error ${configId ? 'updating' : 'adding'} configuration: ${err.message}`);
    return null;
  } finally {
    // Reset loading state
    editingConfiguration.value[itemId] = false;
  }
};

// Function to add a new configuration to an item
const addItemConfiguration = async (itemId, label, value) => {
  if (!label || !value) {
    showError('Configuration name and value are required');
    return;
  }

  addingConfiguration.value[itemId] = true;

  try {
    await updateItemConfiguration(itemId, label, value);
    return true;
  } finally {
    addingConfiguration.value[itemId] = false;
  }
};

// Function to check if a configuration label is in the common configurations list
const isInCommonConfigurations = (label) => {
  return label !== 'custom_configuration' && commonConfigurations.includes(label);
};

// Function to handle configuration type change
const handleConfigTypeChange = (label) => {
  // If the user selects custom_configuration, empty the label field to allow custom input
  if (label === 'custom_configuration') {
    newConfig.value.label = '';
  }

  // Clear the value field when changing configuration type
  newConfig.value.value = '';

  // If selecting datacenter config, set default available values
  if (label === 'dedicated_datacenter') {
    // Auto-select the first datacenter in the dropdown
    if (availableDatacenters.length > 0) {
      selectedDatacenters.value[currentEditConfig.value.itemId] = availableDatacenters[0].code;
    }
  }
};

// Function to fetch available configurations for an item
const fetchItemAvailableConfigurations = async (itemId) => {
  if (!apiToken.value || !activeCartId.value || !itemId) {
    showError('Cannot fetch configurations. Missing required information.');
    return;
  }

  // Set loading state
  loadingAvailableConfigurations.value[itemId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configurations`;

    console.log(`Fetching available configurations for item ${itemId}`);

    const response = await fetch(url, {
      method: 'GET',
      headers
    });

    if (!response.ok) {
      if (response.status === 404) {
        // No available configurations endpoint
        showInfo(`No additional configuration options available for this item.`);
        itemAvailableConfigurations.value[itemId] = [];
        return [];
      }

      const errorData = await response.json();
      throw new Error(`Failed to fetch configurations: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const result = await response.json();
    console.log('Available configurations:', result);

    // Store the available configurations
    itemAvailableConfigurations.value[itemId] = result;

    // If there are available configurations, update the common configurations list
    if (result && result.length > 0) {
      // Extract configuration labels and add them to commonConfigurations if not already present
      const configLabels = result.map(config => config.name || config.label);

      configLabels.forEach(label => {
        if (!commonConfigurations.includes(label)) {
          commonConfigurations.push(label);
        }
      });

      showSuccess(`Found ${result.length} available configuration options`);
    } else {
      showInfo('No additional configuration options available');
    }

    return result;
  } catch (err) {
    console.error('Error fetching configurations:', err);
    showError(`Error fetching configurations: ${err.message}`);
    return [];
  } finally {
    // Reset loading state
    loadingAvailableConfigurations.value[itemId] = false;
  }
};

// Function to show info notification
const showInfo = (message) => {
  // We can reuse the success snackbar but with a different color
  successMessage.value = message;
  // Create a temporary div to store the snackbar and change its color
  const tempDiv = document.createElement('div');
  tempDiv.innerHTML = `<style>.v-snackbar.info-snackbar .v-snackbar__wrapper { background-color: #2196F3 !important; }</style>`;
  document.body.appendChild(tempDiv);

  // Add the info class to the snackbar
  const snackbar = document.querySelector('.v-snackbar');
  if (snackbar) {
    snackbar.classList.add('info-snackbar');
  }

  showSuccessSnackbar.value = true;

  // Remove the style after the snackbar is closed
  setTimeout(() => {
    document.body.removeChild(tempDiv);
    if (snackbar) {
      snackbar.classList.remove('info-snackbar');
    }
  }, 3000);
};

// Function to delete a configuration
const deleteItemConfiguration = async (itemId, configId) => {
  if (!apiToken.value || !activeCartId.value || !itemId || !configId) {
    showError('Cannot delete configuration. Missing required information.');
    return;
  }

  // Set loading state for the specific config
  if (!deleteConfigLoading.value[itemId]) {
    deleteConfigLoading.value[itemId] = {};
  }
  deleteConfigLoading.value[itemId][configId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration/${configId}`;

    console.log(`Deleting configuration ${configId} for item ${itemId}`);

    const response = await fetch(url, {
      method: 'DELETE',
      headers
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to delete configuration: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    showSuccess('Configuration deleted successfully!');

    // Refresh cart item details
    await fetchActiveCartDetails();

    return true;
  } catch (err) {
    console.error('Error deleting configuration:', err);
    showError(`Error deleting configuration: ${err.message}`);
    return null;
  } finally {
    // Reset loading state
    if (deleteConfigLoading.value[itemId]) {
      deleteConfigLoading.value[itemId][configId] = false;
    }
  }
};

// Function to open the delete confirmation dialog
const openDeleteConfirmation = (itemId, config) => {
  configToDelete.value = {
    itemId: itemId,
    configId: config.id,
    label: config.label
  };
  confirmDeleteDialog.value = true;
};

// Function to confirm and process deletion
const confirmDelete = async () => {
  const { itemId, configId } = configToDelete.value;
  await deleteItemConfiguration(itemId, configId);
  confirmDeleteDialog.value = false;
};

// Function to get OS icon based on code
const getOSIcon = (code) => {
  const iconMap = {
    'none_64.en': 'mdi-minus-circle',
    'ubuntu': 'mdi-ubuntu',
    'windows': 'mdi-microsoft-windows',
    'centos': 'mdi-linux',
    'debian': 'mdi-debian'
  };

  // Check if code contains any of the keys
  for (const key in iconMap) {
    if (code.includes(key)) {
      return iconMap[key];
    }
  }

  return 'mdi-server-network';
};

// Function to get the current OS for an item
const getItemOS = (itemId) => {
  if (!cartItemDetails.value[itemId] || !cartItemDetails.value[itemId].configurationDetails) {
    return null;
  }

  const osConfig = cartItemDetails.value[itemId].configurationDetails.find(
    config => config.label === 'dedicated_os'
  );

  return osConfig ? osConfig.value : null;
};

// Function to configure OS for a cart item
const configureItemOS = async (itemId, osCode) => {
  if (!apiToken.value || !activeCartId.value || !itemId || !osCode) {
    showError('Cannot configure OS. Missing required information.');
    return;
  }

  // Set loading state
  configuringOS.value[itemId] = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/item/${itemId}/configuration`;

    const payload = {
      label: "dedicated_os",
      value: osCode
    };

    console.log(`Configuring OS for item ${itemId} to ${osCode}:`, payload);

    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(payload)
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to configure OS: ${response.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const result = await response.json();
    showSuccess(`OS configured to ${osCode} successfully!`);
    console.log('OS configuration result:', result);

    // Refresh cart item details
    await fetchActiveCartDetails();

    return result;
  } catch (err) {
    console.error('Error configuring OS:', err);
    showError(`Error configuring OS: ${err.message}`);
    return null;
  } finally {
    // Reset loading state
    configuringOS.value[itemId] = false;
  }
};

// Manual plan code refs
const showManualPlanCodeDialog = ref(false);
const addingManualPlan = ref(false);
const manualPlanCode = ref('');
const manualDuration = ref('P1M');
const manualQuantity = ref(1);

// Function to add server to cart by manually entered plan code
const addServerByPlanCode = async () => {
  if (!activeCartId.value || !manualPlanCode.value || !manualDuration.value || !manualQuantity.value || manualQuantity.value < 1) {
    showError('Please enter a valid plan code, duration and quantity.');
    return;
  }

  addingManualPlan.value = true;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();
    const url = `${apiEndpoint}/order/cart/${activeCartId.value}/eco`;

    console.log('Adding server to cart by manual plan code:', manualPlanCode.value, 'URL:', url);

    const payload = {
      duration: manualDuration.value,
      planCode: manualPlanCode.value,
      pricingMode: "default",
      quantity: parseInt(manualQuantity.value)
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
    showSuccess(`Added ${manualPlanCode.value} to cart successfully!`);
    console.log('Manual server added to cart result:', result);

    // Reset form
    manualPlanCode.value = '';
    manualQuantity.value = 1;
    showManualPlanCodeDialog.value = false;

    // Refresh cart details
    await fetchActiveCartDetails();

    return result;
  } catch (err) {
    console.error('Error adding server to cart:', err);
    showError(`Error adding server to cart: ${err.message}`);
    return null;
  } finally {
    addingManualPlan.value = false;
  }
};

// Add checkout refs
const checkingOut = ref(false);
const checkoutUrl = ref(null);
const showCheckoutDialog = ref(false);

// Add checkout function
const checkoutCart = async () => {
  if (!apiToken.value || !activeCartId.value) {
    showError('Cannot checkout. No active cart selected.');
    return;
  }

  checkingOut.value = true;
  checkoutUrl.value = null;

  try {
    const headers = {
      'Authorization': `Bearer ${apiToken.value}`,
      'Content-Type': 'application/json'
    };

    const apiEndpoint = getApiEndpoint();

    // Step 1: Attempt to assign the cart (but continue even if it fails)
    console.log('Assigning cart before checkout:', activeCartId.value);
    try {
      const assignUrl = `${apiEndpoint}/order/cart/${activeCartId.value}/assign`;

      const assignResponse = await fetch(assignUrl, {
        method: 'POST',
        headers,
        body: JSON.stringify({})
      });

      if (!assignResponse.ok) {
        const errorData = await assignResponse.json();
        console.warn(`Cart assignment warning (continuing anyway): ${assignResponse.status} - ${errorData.message || JSON.stringify(errorData)}`);
      } else {
        console.log('Cart assignment successful');
      }
    } catch (assignError) {
      // Log the assign error but continue with checkout
      console.warn('Cart assignment failed (continuing anyway):', assignError.message);
    }

    // Step 2: Checkout the cart
    const checkoutUrl = `${apiEndpoint}/order/cart/${activeCartId.value}/checkout`;

    const checkoutPayload = {
      autoPayWithPreferredPaymentMethod: false,
      waiveRetractationPeriod: false
    };

    console.log('Checking out cart:', activeCartId.value, 'with payload:', checkoutPayload);

    const checkoutResponse = await fetch(checkoutUrl, {
      method: 'POST',
      headers,
      body: JSON.stringify(checkoutPayload)
    });

    if (!checkoutResponse.ok) {
      const errorData = await checkoutResponse.json();
      throw new Error(`Failed to checkout cart: ${checkoutResponse.status} - ${errorData.message || JSON.stringify(errorData)}`);
    }

    const checkoutResult = await checkoutResponse.json();
    console.log('Checkout successful:', checkoutResult);

    if (checkoutResult.url) {
      checkoutUrl.value = checkoutResult.url;
      showCheckoutDialog.value = true;
      showSuccess('Checkout successful! Please use the payment link to complete your order.');
    } else {
      showError('Checkout completed but no payment URL was provided.');
    }

    // Refresh cart list and details
    await fetchCart();

    return checkoutResult;
  } catch (err) {
    console.error('Error during checkout process:', err);
    showError(`Checkout error: ${err.message}`);
    return null;
  } finally {
    checkingOut.value = false;
  }
};

// Function to copy text to clipboard
const copyToClipboard = (text) => {
  if (!text) return;

  navigator.clipboard.writeText(text)
    .then(() => {
      showSuccess('URL copied to clipboard!');
    })
    .catch((err) => {
      console.error('Could not copy text: ', err);
      showError('Failed to copy URL');
    });
};
</script>
