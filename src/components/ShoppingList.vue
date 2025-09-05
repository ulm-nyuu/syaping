<template>
  <div class="shopping-container">
    <!-- Header -->
    <header class="header">
      <div class="header-content">
        <h1 class="title">
          <span class="icon">🛒</span>
          Shopping List
        </h1>
        <div class="stats">
          <span class="stat">{{ items.length }} items</span>
          <span class="stat purchased">{{ purchasedCount }} done</span>
        </div>
      </div>
    </header>

    <!-- Search and Filter -->
    <div class="search-section">
      <div class="search-container">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search items..."
          class="search-input"
        />
        <span class="search-icon">🔍</span>
      </div>
      
      <div class="filter-tabs">
        <button
          v-for="filter in filters"
          :key="filter.value"
          @click="activeFilter = filter.value"
          :class="['filter-tab', { active: activeFilter === filter.value }]"
        >
          {{ filter.label }}
        </button>
      </div>
    </div>

    <!-- Add Item Form -->
    <div class="add-item-section">
      <form @submit.prevent="addItem" class="add-form">
        <input
          v-model="newItem.name"
          type="text"
          placeholder="Add new item..."
          class="add-input"
          required
        />
        <div class="quantity-controls">
          <button type="button" @click="decreaseQuantity" class="qty-btn">-</button>
          <span class="quantity">{{ newItem.quantity }}</span>
          <button type="button" @click="increaseQuantity" class="qty-btn">+</button>
        </div>
        <button type="submit" class="add-button">Add</button>
      </form>
    </div>

    <!-- Items List -->
    <div class="items-section">
      <TransitionGroup name="list" tag="div" class="items-list">
        <div
          v-for="item in filteredItems"
          :key="item.id"
          :class="['item-card', { purchased: item.purchased }]"
        >
          <div class="item-content" @click="togglePurchased(item.id)">
            <div class="item-check">
              <div :class="['checkbox', { checked: item.purchased }]">
                <span v-if="item.purchased" class="checkmark">✓</span>
              </div>
            </div>
            
            <div class="item-details">
              <h3 class="item-name">{{ item.name }}</h3>
              <div class="item-meta">
                <span class="category">{{ item.category }}</span>
                <span class="quantity">Qty: {{ item.quantity }}</span>
              </div>
            </div>

            <div class="item-actions">
              <button @click.stop="editItem(item)" class="action-btn edit">
                ✏️
              </button>
              <button @click.stop="deleteItem(item.id)" class="action-btn delete">
                🗑️
              </button>
            </div>
          </div>
        </div>
      </TransitionGroup>
      
      <div v-if="filteredItems.length === 0" class="empty-state">
        <span class="empty-icon">📝</span>
        <h3>{{ getEmptyStateMessage() }}</h3>
        <p>{{ getEmptyStateSubtext() }}</p>
      </div>
    </div>

    <!-- Edit Modal -->
    <div v-if="editingItem" class="modal-overlay" @click="cancelEdit">
      <div class="modal" @click.stop>
        <h3 class="modal-title">Edit Item</h3>
        <form @submit.prevent="saveEdit" class="edit-form">
          <input
            v-model="editingItem.name"
            type="text"
            class="modal-input"
            required
          />
          <select v-model="editingItem.category" class="modal-select">
            <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
          </select>
          <div class="modal-quantity">
            <label>Quantity:</label>
            <div class="quantity-controls">
              <button type="button" @click="editingItem.quantity--" class="qty-btn">-</button>
              <span class="quantity">{{ editingItem.quantity }}</span>
              <button type="button" @click="editingItem.quantity++" class="qty-btn">+</button>
            </div>
          </div>
          <div class="modal-actions">
            <button type="button" @click="cancelEdit" class="modal-btn cancel">Cancel</button>
            <button type="submit" class="modal-btn save">Save</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'

interface ShoppingItem {
  id: number
  name: string
  category: string
  quantity: number
  purchased: boolean
  createdAt: Date
}

// Reactive state
const items = ref<ShoppingItem[]>([])
const searchQuery = ref('')
const activeFilter = ref<'all' | 'pending' | 'purchased'>('all')
const editingItem = ref<ShoppingItem | null>(null)
const newItem = ref({
  name: '',
  quantity: 1,
  category: 'Groceries'
})

// Constants
const categories = ['Groceries', 'Electronics', 'Clothing', 'Home', 'Personal', 'Other']
const filters = [
  { value: 'all', label: 'All' },
  { value: 'pending', label: 'Pending' },
  { value: 'purchased', label: 'Done' }
]

// Computed properties
const purchasedCount = computed(() => 
  items.value.filter(item => item.purchased).length
)

const filteredItems = computed(() => {
  let filtered = items.value

  // Filter by search query
  if (searchQuery.value) {
    filtered = filtered.filter(item =>
      item.name.toLowerCase().includes(searchQuery.value.toLowerCase())
    )
  }

  // Filter by status
  if (activeFilter.value === 'pending') {
    filtered = filtered.filter(item => !item.purchased)
  } else if (activeFilter.value === 'purchased') {
    filtered = filtered.filter(item => item.purchased)
  }

  return filtered.sort((a, b) => {
    if (a.purchased !== b.purchased) {
      return a.purchased ? 1 : -1
    }
    return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
  })
})

// Methods
const addItem = () => {
  if (!newItem.value.name.trim()) return

  const item: ShoppingItem = {
    id: Date.now(),
    name: newItem.value.name.trim(),
    category: newItem.value.category,
    quantity: newItem.value.quantity,
    purchased: false,
    createdAt: new Date()
  }

  items.value.unshift(item)
  newItem.value.name = ''
  newItem.value.quantity = 1
  saveToStorage()
}

const deleteItem = (id: number) => {
  items.value = items.value.filter(item => item.id !== id)
  saveToStorage()
}

const togglePurchased = (id: number) => {
  const item = items.value.find(item => item.id === id)
  if (item) {
    item.purchased = !item.purchased
    saveToStorage()
  }
}

const editItem = (item: ShoppingItem) => {
  editingItem.value = { ...item }
}

const saveEdit = () => {
  if (!editingItem.value) return

  const index = items.value.findIndex(item => item.id === editingItem.value!.id)
  if (index !== -1) {
    items.value[index] = { ...editingItem.value }
    saveToStorage()
  }
  editingItem.value = null
}

const cancelEdit = () => {
  editingItem.value = null
}

const increaseQuantity = () => {
  newItem.value.quantity++
}

const decreaseQuantity = () => {
  if (newItem.value.quantity > 1) {
    newItem.value.quantity--
  }
}

const getEmptyStateMessage = () => {
  if (searchQuery.value) return 'No items found'
  if (activeFilter.value === 'pending') return 'No pending items'
  if (activeFilter.value === 'purchased') return 'No completed items'
  return 'Your shopping list is empty'
}

const getEmptyStateSubtext = () => {
  if (searchQuery.value) return 'Try adjusting your search terms'
  if (activeFilter.value === 'pending') return 'Great job! All items are done'
  if (activeFilter.value === 'purchased') return 'Start shopping to see completed items here'
  return 'Add your first item to get started'
}

// Storage methods
const saveToStorage = () => {
  localStorage.setItem('shopping-list', JSON.stringify(items.value))
}

const loadFromStorage = () => {
  const stored = localStorage.getItem('shopping-list')
  if (stored) {
    try {
      const parsed = JSON.parse(stored)
      items.value = parsed.map((item: any) => ({
        ...item,
        createdAt: new Date(item.createdAt)
      }))
    } catch (error) {
      console.error('Error loading from storage:', error)
    }
  }
}

// Lifecycle
onMounted(() => {
  loadFromStorage()
})

// Auto-save on changes
watch(items, saveToStorage, { deep: true })
</script>

<style scoped>
.shopping-container {
  width: 100%;
  max-width: 425px;
  min-height: 100vh;
  background: white;
  display: flex;
  flex-direction: column;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.1);
}

/* Header */
.header {
  background: linear-gradient(135deg, #3B82F6, #1D4ED8);
  color: white;
  padding: 1.5rem 1rem 1rem;
  position: sticky;
  top: 0;
  z-index: 10;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.title {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0;
}

.icon {
  font-size: 1.8rem;
}

.stats {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.25rem;
}

.stat {
  font-size: 0.875rem;
  opacity: 0.9;
}

.stat.purchased {
  color: #10B981;
  font-weight: 600;
}

/* Search Section */
.search-section {
  padding: 1rem;
  background: #F8FAFC;
  border-bottom: 1px solid #E2E8F0;
}

.search-container {
  position: relative;
  margin-bottom: 1rem;
}

.search-input {
  width: 100%;
  padding: 0.75rem 1rem 0.75rem 2.5rem;
  border: 2px solid #E2E8F0;
  border-radius: 12px;
  font-size: 1rem;
  background: white;
  transition: border-color 0.2s;
}

.search-input:focus {
  outline: none;
  border-color: #3B82F6;
}

.search-icon {
  position: absolute;
  left: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  opacity: 0.5;
}

.filter-tabs {
  display: flex;
  gap: 0.5rem;
}

.filter-tab {
  flex: 1;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 8px;
  background: white;
  color: #64748B;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.filter-tab.active {
  background: #3B82F6;
  color: white;
}

.filter-tab:hover:not(.active) {
  background: #F1F5F9;
}

/* Add Item Section */
.add-item-section {
  padding: 1rem;
  background: white;
  border-bottom: 1px solid #E2E8F0;
}

.add-form {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.add-input {
  flex: 1;
  padding: 0.75rem 1rem;
  border: 2px solid #E2E8F0;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s;
}

.add-input:focus {
  outline: none;
  border-color: #3B82F6;
}

.quantity-controls {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: #F8FAFC;
  border-radius: 8px;
  padding: 0.25rem;
}

.qty-btn {
  width: 32px;
  height: 32px;
  border: none;
  border-radius: 6px;
  background: white;
  color: #3B82F6;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.qty-btn:hover {
  background: #3B82F6;
  color: white;
}

.quantity {
  min-width: 2rem;
  text-align: center;
  font-weight: 600;
  color: #374151;
}

.add-button {
  padding: 0.75rem 1.5rem;
  background: #10B981;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.add-button:hover {
  background: #059669;
}

/* Items Section */
.items-section {
  flex: 1;
  padding: 1rem;
  overflow-y: auto;
}

.items-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.item-card {
  background: white;
  border-radius: 12px;
  border: 2px solid #F1F5F9;
  overflow: hidden;
  transition: all 0.3s ease;
}

.item-card:hover {
  border-color: #E2E8F0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.item-card.purchased {
  opacity: 0.7;
  background: #F8FAFC;
}

.item-content {
  display: flex;
  align-items: center;
  padding: 1rem;
  cursor: pointer;
  min-height: 60px;
}

.item-check {
  margin-right: 1rem;
}

.checkbox {
  width: 24px;
  height: 24px;
  border: 2px solid #CBD5E1;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: white;
  transition: all 0.2s;
}

.checkbox.checked {
  background: #10B981;
  border-color: #10B981;
}

.checkmark {
  color: white;
  font-weight: bold;
  font-size: 0.875rem;
}

.item-details {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 600;
  margin: 0 0 0.25rem 0;
  color: #1F2937;
  word-break: break-word;
}

.item-card.purchased .item-name {
  text-decoration: line-through;
  color: #6B7280;
}

.item-meta {
  display: flex;
  gap: 1rem;
  font-size: 0.875rem;
  color: #6B7280;
}

.category {
  background: #EEF2FF;
  color: #3730A3;
  padding: 0.125rem 0.5rem;
  border-radius: 4px;
  font-weight: 500;
}

.item-actions {
  display: flex;
  gap: 0.5rem;
  opacity: 0.6;
  transition: opacity 0.2s;
}

.item-card:hover .item-actions {
  opacity: 1;
}

.action-btn {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 8px;
  background: #F8FAFC;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.action-btn:hover {
  background: #E2E8F0;
}

.action-btn.delete:hover {
  background: #FEE2E2;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 3rem 1rem;
  color: #6B7280;
}

.empty-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
  opacity: 0.5;
}

.empty-state h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.25rem;
  color: #374151;
}

.empty-state p {
  margin: 0;
  font-size: 0.875rem;
}

/* Modal */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 1rem;
}

.modal {
  background: white;
  border-radius: 16px;
  padding: 1.5rem;
  width: 100%;
  max-width: 400px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.modal-title {
  margin: 0 0 1.5rem 0;
  font-size: 1.25rem;
  font-weight: 700;
  color: #1F2937;
}

.edit-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.modal-input,
.modal-select {
  padding: 0.75rem 1rem;
  border: 2px solid #E2E8F0;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s;
}

.modal-input:focus,
.modal-select:focus {
  outline: none;
  border-color: #3B82F6;
}

.modal-quantity {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.modal-quantity label {
  font-weight: 600;
  color: #374151;
}

.modal-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 0.5rem;
}

.modal-btn {
  flex: 1;
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.modal-btn.cancel {
  background: #F3F4F6;
  color: #374151;
}

.modal-btn.cancel:hover {
  background: #E5E7EB;
}

.modal-btn.save {
  background: #3B82F6;
  color: white;
}

.modal-btn.save:hover {
  background: #2563EB;
}

/* Animations */
.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}

.list-enter-from {
  opacity: 0;
  transform: translateY(-20px);
}

.list-leave-to {
  opacity: 0;
  transform: translateX(20px);
}

/* Responsive adjustments */
@media (max-width: 480px) {
  .add-form {
    flex-wrap: wrap;
  }
  
  .add-input {
    min-width: 100%;
    margin-bottom: 0.5rem;
  }
  
  .quantity-controls,
  .add-button {
    margin-top: 0.5rem;
  }
  
  .item-meta {
    flex-direction: column;
    gap: 0.25rem;
  }
}
</style>