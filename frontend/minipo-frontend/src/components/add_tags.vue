<!-- ProductInventory.vue -->
<template>
  <div class="container mt-4">
    <!-- Add New Product Card -->
    <div class="row mb-4">
      <div class="col-12">
        <div class="card">
          <div class="card-body">
            <div class="d-flex gap-3">
              <input 
                type="text" 
                class="form-control w-auto" 
                v-model="newProductName"
                placeholder="Enter new product name"
                :disabled="addingProduct"
              >
              <button 
                class="btn btn-primary d-flex align-items-center gap-2"
                @click="addNewProduct"
                :disabled="addingProduct"
              >
                <span v-if="addingProduct" class="spinner-border spinner-border-sm" role="status"></span>
                <span>{{ addingProduct ? 'Adding...' : 'Add Product' }}</span>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Products Grid and Table Layout -->
    <div class="row">
      <!-- Product Cards Grid -->
      <div class="col-md-4">
        <div class="row">
          <div v-for="(product, productId) in inventory" :key="productId" class="col-12 mb-3">
            <div 
              class="card"
              :class="{ 'border-primary': selectedProduct === productId }"
              style="cursor: pointer;"
              @click="selectProduct(productId)"
            >
              <div class="card-body">
                <h5 class="card-title text-capitalize">{{ product.name }}</h5>
                <p class="card-text mb-3">
                  Total Tags: {{ product.total_quantity }}
                </p>
                <button 
                  @click.stop="addTag(productId)"
                  class="btn btn-primary"
                  :disabled="isReading"
                >
                  <span v-if="isReading" class="spinner-border spinner-border-sm me-2" role="status"></span>
                  {{ isReading ? 'Reading Tag...' : 'Add Tag' }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- RFID Tags Table -->
      <div class="col-md-8">
        <div class="card">
          <div class="card-header">
            <h5 class="card-title mb-0">
              {{ selectedProduct ? `RFID Tags for ${inventory[selectedProduct]?.name}` : 'Select a product to view tags' }}
            </h5>
          </div>
          <div class="card-body">
            <div v-if="selectedProduct && inventory[selectedProduct]">
              <table class="table">
                <thead>
                  <tr>
                    <th>Tag ID</th>
                    <th>Raw ID</th>
                    <th>Added Date</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(tagInfo, tagId) in inventory[selectedProduct].rfid_tags" :key="tagId">
                    <td>{{ tagId }}</td>
                    <td>{{ tagInfo.raw_id }}</td>
                    <td>{{ formatDate(tagInfo.added) }}</td>
                  </tr>
                  <tr v-if="!inventory[selectedProduct].rfid_tags || Object.keys(inventory[selectedProduct].rfid_tags).length === 0">
                    <td colspan="3" class="text-center">No tags available</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div v-else class="text-center text-muted py-4">
              Select a product to view its RFID tags
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Error Alert -->
    <div 
      v-if="error" 
      class="alert alert-danger alert-dismissible fade show mt-3" 
      role="alert"
    >
      {{ error }}
      <button 
        type="button" 
        class="btn-close" 
        @click="error = ''" 
        aria-label="Close"
      ></button>
    </div>

    <!-- Loading Overlay -->
    <div 
      v-if="loading" 
      class="position-fixed top-0 start-0 w-100 h-100 d-flex justify-content-center align-items-center"
      style="background: rgba(255,255,255,0.8); z-index: 1000;"
    >
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </div>
</template>

<script>
import { initializeApp } from 'firebase/app';
import { getDatabase, ref, get, set } from 'firebase/database';

// Firebase configuration
const firebaseConfig = {
  databaseURL: 'https://iot-sem5-minipo-default-rtdb.asia-southeast1.firebasedatabase.app'
};

// Raspberry Pi Pico W IP address
const PICO_IP = '192.168.29.182';

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const db = getDatabase(app);

export default {
  name: 'ProductInventory',
  data() {
    return {
      inventory: {},
      isReading: false,
      error: '',
      loading: false,
      selectedProduct: null,
      newProductName: '',
      addingProduct: false
    };
  },
  methods: {
    async fetchInventory() {
      this.loading = true;
      try {
        const inventoryRef = ref(db, 'inventory');
        const snapshot = await get(inventoryRef);
        if (snapshot.exists()) {
          this.inventory = snapshot.val();
        }
      } catch (error) {
        console.error('Error fetching inventory:', error);
        this.error = 'Failed to fetch inventory data';
      } finally {
        this.loading = false;
      }
    },
    formatDate(timestamp) {
      return new Date(timestamp * 1000).toLocaleString();
    },
    selectProduct(productId) {
      this.selectedProduct = productId;
    },
    async addNewProduct() {
      if (!this.newProductName.trim()) {
        this.error = 'Product name cannot be empty';
        return;
      }

      this.addingProduct = true;
      try {
        const productRef = ref(db, `inventory/${this.newProductName.toLowerCase()}`);
        await set(productRef, {
          name: this.newProductName.toLowerCase(),
          rfid_tags: {},
          total_quantity: 0
        });
        await this.fetchInventory();
        this.newProductName = '';
        this.error = '';
      } catch (error) {
        console.error('Error adding product:', error);
        this.error = 'Failed to add new product';
      } finally {
        this.addingProduct = false;
      }
    },
    async addTag(productId) {
      if (this.isReading) return;
      
      this.isReading = true;
      this.error = '';
      
      try {
        const response = await fetch(
          `http://${PICO_IP}/read_tag?product=${encodeURIComponent(productId)}`
        );
        const data = await response.json();
        
        if (data.status === 'success') {
          await this.fetchInventory();
        } else {
          this.error = data.message || 'Failed to read tag';
        }
      } catch (error) {
        console.error('Error reading tag:', error);
        this.error = 'Failed to communicate with RFID reader';
      } finally {
        this.isReading = false;
      }
    }
  },
  mounted() {
    this.fetchInventory();
  }
};
</script>

<style scoped>
.card {
  transition: all 0.3s ease;
}
.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}
</style>