<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate your budget across low-stock items to optimize inventory levels</p>
    </div>

    <div v-if="submittedOrder" class="success-banner">
      <div class="success-content">
        <span class="success-text">
          Restocking order {{ submittedOrder.order_number }} placed successfully! Expected delivery: {{ formatDate(submittedOrder.expected_delivery) }}
        </span>
        <button class="btn-secondary" @click="submittedOrder = null">Place Another Order</button>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="budget-row">
          <label class="budget-label">Available Budget</label>
          <div class="slider-row">
            <input
              type="range"
              min="0"
              max="1000000"
              step="1000"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
        </div>
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Total Budget</div>
          <div class="stat-value" style="color: #2563eb;">{{ formatCurrency(budget) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Allocated</div>
          <div class="stat-value" style="color: #059669;">{{ formatCurrency(allocatedAmount) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Remaining</div>
          <div class="stat-value">{{ formatCurrency(budget - allocatedAmount) }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Items Selected</div>
          <div class="stat-value" style="color: #2563eb;">{{ selectedItems.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="badge info">{{ recommendations.length }}</span>
        </div>
        <div v-if="recommendations.length === 0" class="loading">
          No low-stock items found
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Priority</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Warehouse</th>
                <th>Stock</th>
                <th>Reorder Pt</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Est. Cost</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :style="isSelected(item) ? '' : 'opacity: 0.4'"
              >
                <td>
                  <span :class="['badge', getPriorityClass(item.priority_label)]">
                    {{ item.priority_label }}
                  </span>
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.current_stock }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>{{ item.quantity_to_order }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>{{ formatCurrency(item.estimated_cost) }}</td>
                <td>
                  <span v-if="item.demand_trend" :class="['badge', item.demand_trend]">
                    {{ item.demand_trend }}
                  </span>
                  <span v-else>—</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="selectedItems.length > 0 && !submittedOrder" class="order-action">
          <button
            class="btn-place-order"
            :disabled="submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(250000)
    const recommendations = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submittedOrder = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const selectedItems = computed(() => {
      let running = 0
      return recommendations.value.filter(item => {
        if (running + item.estimated_cost <= budget.value) {
          running += item.estimated_cost
          return true
        }
        return false
      })
    })

    const allocatedAmount = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    )

    const isSelected = (item) => {
      return selectedItems.value.some(s => s.sku === item.sku)
    }

    const getPriorityClass = (priority) => {
      const map = {
        'High': 'danger',
        'Medium': 'warning',
        'Low': 'info'
      }
      return map[priority] || 'info'
    }

    const formatCurrency = (value) =>
      '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })

    const formatDate = (dateStr) =>
      new Date(dateStr).toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })

    const placeOrder = async () => {
      submitting.value = true
      try {
        const orderData = {
          items: selectedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.quantity_to_order,
            unit_price: item.unit_cost
          })),
          total_value: allocatedAmount.value
        }
        const result = await api.submitRestockingOrder(orderData)
        submittedOrder.value = result
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      budget,
      recommendations,
      loading,
      error,
      submitting,
      submittedOrder,
      selectedItems,
      allocatedAmount,
      isSelected,
      getPriorityClass,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-row {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: #e2e8f0;
  border-radius: 4px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.order-action {
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.btn-place-order {
  width: 100%;
  padding: 0.875rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
}

.success-text {
  color: #065f46;
  font-size: 0.938rem;
  font-weight: 500;
}

.btn-secondary {
  padding: 0.5rem 1rem;
  background: white;
  color: #065f46;
  border: 1px solid #6ee7b7;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  white-space: nowrap;
}

.btn-secondary:hover {
  background: #ecfdf5;
}
</style>
