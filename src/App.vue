<script setup lang="ts">
import {ref, watch} from 'vue';

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

const productsStorageKey = 'product-management-products';
const persistenceError = ref<string | null>(null);

const isProduct = (value: unknown): value is Product => {
  if (typeof value !== 'object' || value === null) {
    return false;
  }

  const product = value as Record<string, unknown>;
  return Number.isFinite(product.id)
    && typeof product.name === 'string'
    && Number.isFinite(product.price)
    && typeof product.description === 'string';
};

const loadProducts = (): Product[] => {
  try {
    const savedProducts = window.localStorage.getItem(productsStorageKey);
    if (savedProducts === null) {
      return [];
    }

    const parsedProducts: unknown = JSON.parse(savedProducts);
    if (!Array.isArray(parsedProducts) || !parsedProducts.every(isProduct)) {
      throw new Error('Saved product data has an invalid format.');
    }

    return parsedProducts;
  } catch (error) {
    console.error('Unable to load saved products.', error);
    persistenceError.value = 'Saved products could not be loaded. Changes may be lost when you reload.';
    return [];
  }
};

const products = ref<Product[]>(loadProducts());
watch(products, (savedProducts) => {
  try {
    window.localStorage.setItem(productsStorageKey, JSON.stringify(savedProducts));
    persistenceError.value = null;
  } catch (error) {
    console.error('Unable to save products.', error);
    persistenceError.value = 'Products could not be saved in this browser. Changes may be lost when you reload.';
  }
}, { deep: true });

const newProduct = ref<Omit<Product, 'id'>>({
  name: '',
  price: 0,
  description: ''
});
const editingProductId = ref<number | null>(null);

const resetForm = () => {
  newProduct.value = { name: '', price: 0, description: '' };
  editingProductId.value = null;
};

const addProduct = () => {
  if (!newProduct.value.name || !newProduct.value.price) {
    alert('Please fill in all fields with valid values.');
    return;
  }

  if (editingProductId.value !== null) {
    const index = products.value.findIndex(p => p.id === editingProductId.value);
    if (index !== -1) {
      products.value[index] = { id: editingProductId.value, ...newProduct.value };
    }
    resetForm();
    return;
  }

  products.value.push({ id: Date.now(), ...newProduct.value });
  resetForm();
};

const startEdit = (product: Product) => {
  editingProductId.value = product.id;
  newProduct.value = {
    name: product.name,
    price: product.price,
    description: product.description,
  };
};

const cancelEdit = () => {
  resetForm();
};

const deleteProduct = (productId: number) => {
  products.value = products.value.filter(product => product.id !== productId);
  if (editingProductId.value === productId) {
    resetForm();
  }
};

</script>

<template>
  <div class="min-h-screen bg-gray-100 flex items-center justify-center p-4">
    <div class="max-w-4xl w-full bg-white rounded-lg shadow-lg p-8">
      <h1 class="text-4xl font-bold text-center text-gray-800 mb-8">Product Management</h1>

      <!-- Add Product Section -->
      <div class="mb-8">
        <h2 class="text-2xl font-semibold text-gray-700 mb-4">
          {{ editingProductId !== null ? 'Edit Product' : 'Add New Product' }}
        </h2>
        <form @submit.prevent="addProduct" class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Product Name</label>
            <input
              v-model="newProduct.name"
              type="text"
              placeholder="Enter product name"
              class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              required
            />
          </div>
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Price</label>
            <input
              v-model="newProduct.price"
              type="number"
              step="0.01"
              placeholder="Enter price"
              class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              required
            />
          </div>
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
            <textarea
              v-model="newProduct.description"
              placeholder="Enter product description"
              rows="3"
              class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            ></textarea>
          </div>
          <div class="flex gap-3">
            <button
              type="submit"
              class="flex-1 bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 transition duration-200"
            >
              {{ editingProductId !== null ? 'Save Changes' : 'Add Product' }}
            </button>
            <button
              v-if="editingProductId !== null"
              type="button"
              @click="cancelEdit"
              class="flex-1 bg-gray-200 text-gray-700 py-2 px-4 rounded-md hover:bg-gray-300 focus:outline-none focus:ring-2 focus:ring-gray-400 focus:ring-offset-2 transition duration-200"
            >
              Cancel
            </button>
          </div>
        </form>
      </div>

      <!-- Products List Section -->
      <div>
        <h2 class="text-2xl font-semibold text-gray-700 mb-4">Products</h2>
        <p v-if="persistenceError" role="alert" class="mb-4 rounded-md bg-amber-100 p-3 text-amber-800">
          {{ persistenceError }}
        </p>
        <div v-if="products.length === 0" class="text-center text-gray-500 py-8">
          No products added yet.
        </div>
        <div v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          <div
            v-for="product in products"
            :key="product.id"
            class="bg-gray-50 border border-gray-200 rounded-lg p-4 shadow-sm hover:shadow-md transition duration-200"
          >
            <div class="flex items-start justify-between gap-3 mb-3">
              <div>
                <h3 class="text-lg font-semibold text-gray-800">{{ product.name }}</h3>
                <p class="text-sm text-gray-500">ID: {{ product.id }}</p>
              </div>
              <div class="flex gap-2">
                <button
                  @click="startEdit(product)"
                  class="px-3 py-1 text-sm text-white bg-yellow-500 rounded-md hover:bg-yellow-600 transition duration-200"
                >
                  Edit
                </button>
                <button
                  @click="deleteProduct(product.id)"
                  class="px-3 py-1 text-sm text-white bg-red-500 rounded-md hover:bg-red-600 transition duration-200"
                >
                  Delete
                </button>
              </div>
            </div>
            <p class="text-green-600 font-bold text-xl mb-2">${{ product.price.toFixed(2) }}</p>
            <p class="text-gray-600">{{ product.description }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
