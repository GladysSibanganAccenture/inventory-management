<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Manage inventory replenishment based on reorder points and available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card danger">
          <div class="stat-label">Low Stock Items</div>
          <div class="stat-value">{{ lowStockItems.length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Items Within Budget</div>
          <div class="stat-value">{{ recommendedItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Estimated Cost</div>
          <div class="stat-value">{{ formatCurrency(estimatedCost) }}</div>
        </div>
      </div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <label class="budget-label" for="budget-slider">Restocking Budget</label>
          <div class="slider-row">
            <span class="slider-min">$0</span>
            <input
              id="budget-slider"
              type="range"
              min="0"
              max="50000"
              step="500"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="slider-max">$50,000</span>
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
        </div>
      </div>

      <div v-if="successBanner" class="success-banner">
        <strong>Order Submitted Successfully</strong>
        <div class="banner-details">
          <span>Order Number: <strong>{{ successBanner.order_number }}</strong></span>
          <span>Expected Delivery: <strong>{{ formatDate(successBanner.expected_delivery) }}</strong></span>
        </div>
      </div>

      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restock Items ({{ recommendedItems.length }})</h3>
          <button
            class="btn btn-primary"
            :disabled="recommendedItems.length === 0 || submitted"
            @click="placeOrder"
          >
            {{ submitted ? 'Order Placed' : 'Place Order' }}
          </button>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items can be restocked within the current budget. Increase the budget to see recommendations.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th class="col-num">Qty to Restock</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td><span class="badge info">{{ item.category }}</span></td>
                <td>{{ item.warehouse }}</td>
                <td class="col-num">{{ item.restock_qty.toLocaleString() }}</td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num"><strong>{{ formatCurrency(item.line_total) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="summary-row">
                <td colspan="6" class="summary-label">Total Estimated Cost</td>
                <td class="col-num summary-total">
                  <strong>{{ formatCurrency(estimatedCost) }}</strong>
                  <span class="budget-remaining"> / {{ formatCurrency(budget) }} budget</span>
                </td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>

      <div v-if="lowStockItems.length > recommendedItems.length" class="card">
        <div class="card-header">
          <h3 class="card-title">Items Exceeding Budget ({{ lowStockItems.length - recommendedItems.length }})</h3>
          <span class="badge warning">Not included</span>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th class="col-num">Qty to Restock</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in excludedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td><span class="badge info">{{ item.category }}</span></td>
                <td>{{ item.warehouse }}</td>
                <td class="col-num">{{ item.restock_qty.toLocaleString() }}</td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num danger-text"><strong>{{ formatCurrency(item.line_total) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(false)
    const error = ref(null)
    const submitError = ref(null)
    const submitted = ref(false)
    const successBanner = ref(null)

    const inventory = ref([])
    const budget = ref(10000)

    const lowStockItems = computed(() => {
      return inventory.value
        .filter(item => item.quantity_on_hand <= item.reorder_point)
        .map(item => {
          const restock_qty = (item.reorder_point * 2) - item.quantity_on_hand
          const line_total = restock_qty * item.unit_cost
          return {
            sku: item.sku,
            name: item.name,
            category: item.category,
            warehouse: item.warehouse,
            quantity_on_hand: item.quantity_on_hand,
            reorder_point: item.reorder_point,
            unit_cost: item.unit_cost,
            restock_qty,
            line_total,
            urgency: item.quantity_on_hand - item.reorder_point
          }
        })
        .sort((a, b) => a.urgency - b.urgency)
    })

    const recommendedItems = computed(() => {
      let remaining = budget.value
      const result = []
      for (const item of lowStockItems.value) {
        if (item.line_total <= remaining) {
          result.push(item)
          remaining -= item.line_total
        }
      }
      return result
    })

    const excludedItems = computed(() => {
      const recommendedSkus = new Set(recommendedItems.value.map(i => i.sku))
      return lowStockItems.value.filter(i => !recommendedSkus.has(i.sku))
    })

    const estimatedCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.line_total, 0)
    })

    const formatCurrency = (value) => {
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 2 })
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inv] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventory.value = inv
      } catch (err) {
        error.value = 'Failed to load inventory data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitError.value = null
      const items = recommendedItems.value.map(item => ({
        sku: item.sku,
        name: item.name,
        category: item.category,
        warehouse: item.warehouse,
        quantity: item.restock_qty,
        unit_cost: item.unit_cost,
        line_total: item.line_total
      }))
      try {
        const result = await api.submitRestockingOrder({
          items,
          total_value: estimatedCost.value
        })
        successBanner.value = result
        submitted.value = true
      } catch (err) {
        submitError.value = 'Failed to submit restocking order: ' + err.message
        console.error(err)
      }
    }

    watch(budget, () => {
      successBanner.value = null
      submitted.value = false
      submitError.value = null
    })

    onMounted(() => loadData())

    return {
      loading,
      error,
      submitError,
      submitted,
      successBanner,
      budget,
      lowStockItems,
      recommendedItems,
      excludedItems,
      estimatedCost,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-min,
.slider-max {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #2563eb;
  min-width: 100px;
  text-align: right;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.banner-details {
  display: flex;
  gap: 2rem;
  font-size: 0.875rem;
}

.btn {
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s ease;
}

.btn-primary {
  background: #2563eb;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.restock-table {
  table-layout: auto;
  width: 100%;
}

.col-num {
  text-align: right;
}

.summary-row {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.summary-label {
  text-align: right;
  font-weight: 600;
  color: #475569;
  font-size: 0.875rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  padding-right: 0.75rem;
}

.summary-total {
  text-align: right;
  font-size: 1rem;
  color: #0f172a;
}

.budget-remaining {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 400;
  margin-left: 0.25rem;
}

.danger-text {
  color: #dc2626;
}
</style>
