<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Budget-based restock recommendations from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-control">
          <label class="budget-label" for="budget-slider">Restock Budget</label>
          <div class="slider-row">
            <span class="budget-min">$0</span>
            <input
              id="budget-slider"
              type="range"
              min="0"
              max="500000"
              step="5000"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="budget-max">$500,000</span>
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
        </div>
      </div>

      <!-- Success message -->
      <div v-if="successMessage" class="success-message">
        <strong>Order placed successfully.</strong>
        Order {{ successMessage.order_number }} submitted.
        Expected delivery: {{ formatDate(successMessage.expected_delivery) }}.
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking ({{ recommendations.length }} items)</h3>
          <div class="budget-summary">
            <span class="summary-item">
              Selected Total: <strong>{{ formatCurrency(selectedTotal) }}</strong>
            </span>
            <span class="summary-separator">|</span>
            <span class="summary-item" :class="{ 'over-budget': isOverBudget }">
              Remaining: <strong>{{ formatCurrency(budget - selectedTotal) }}</strong>
            </span>
            <span v-if="isOverBudget" class="budget-warning">Over budget</span>
          </div>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items with growing demand found, or budget is too low for any restock.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Trend</th>
                <th>Demand Gap</th>
                <th>Recommended Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.gap }}</td>
                <td>
                  <input
                    type="number"
                    class="qty-input"
                    :value="item.quantity"
                    :min="0"
                    :max="item.gap"
                    @input="updateQuantity(item.item_sku, $event)"
                  />
                </td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.quantity * item.unit_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="card-footer">
          <button
            class="place-order-btn"
            :disabled="!canPlaceOrder"
            @click="placeOrder"
          >
            {{ placingOrder ? 'Submitting...' : 'Place Order' }}
          </button>
          <span v-if="orderError" class="order-error">{{ orderError }}</span>
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
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(100000)
    const placingOrder = ref(false)
    const orderError = ref(null)
    const successMessage = ref(null)

    // Per-item user-edited quantities: { [sku]: quantity }
    const userQuantities = ref({})

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        forecasts.value = data
      } catch (err) {
        error.value = 'Failed to load demand forecasts'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Eligible items: gap > 0, sorted by gap descending
    const eligibleItems = computed(() => {
      return forecasts.value
        .map(f => ({
          item_sku: f.item_sku,
          item_name: f.item_name,
          trend: f.trend,
          current_demand: f.current_demand,
          forecasted_demand: f.forecasted_demand,
          unit_cost: f.unit_cost,
          gap: Math.max(0, f.forecasted_demand - f.current_demand)
        }))
        .filter(f => f.gap > 0)
        .sort((a, b) => b.gap - a.gap)
    })

    // Greedy allocation based on budget
    const greedyQuantities = computed(() => {
      const result = {}
      let remaining = budget.value
      for (const item of eligibleItems.value) {
        const fullCost = item.gap * item.unit_cost
        if (fullCost <= remaining) {
          result[item.item_sku] = item.gap
          remaining -= fullCost
        } else {
          const qty = Math.floor(remaining / item.unit_cost)
          result[item.item_sku] = qty
          remaining -= qty * item.unit_cost
          if (remaining <= 0) break
        }
      }
      return result
    })

    // Reset user quantities to greedy allocation when budget changes
    watch(greedyQuantities, (newQtys) => {
      userQuantities.value = { ...newQtys }
    }, { immediate: true })

    // Recommendations: eligible items merged with user quantities
    const recommendations = computed(() => {
      return eligibleItems.value
        .filter(item => greedyQuantities.value[item.item_sku] !== undefined)
        .map(item => ({
          ...item,
          quantity: userQuantities.value[item.item_sku] ?? 0
        }))
    })

    const selectedTotal = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + item.quantity * item.unit_cost, 0)
    })

    const isOverBudget = computed(() => selectedTotal.value > budget.value)

    const canPlaceOrder = computed(() => {
      return !placingOrder.value &&
        !isOverBudget.value &&
        recommendations.value.some(item => item.quantity > 0)
    })

    const updateQuantity = (sku, event) => {
      const item = eligibleItems.value.find(i => i.item_sku === sku)
      if (!item) return
      let val = parseInt(event.target.value, 10)
      if (isNaN(val)) val = 0
      val = Math.max(0, Math.min(item.gap, val))
      userQuantities.value = { ...userQuantities.value, [sku]: val }
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      placingOrder.value = true
      orderError.value = null
      successMessage.value = null
      try {
        const items = recommendations.value
          .filter(item => item.quantity > 0)
          .map(item => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.quantity,
            unit_cost: item.unit_cost
          }))
        const result = await api.createRestockOrder({ items, budget: budget.value })
        successMessage.value = result
        // Reset to greedy for current budget
        userQuantities.value = { ...greedyQuantities.value }
      } catch (err) {
        orderError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const formatCurrency = (val) => {
      return val.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatDate = (dateString) => {
      const d = new Date(dateString)
      if (isNaN(d.getTime())) return dateString
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    onMounted(loadForecasts)

    return {
      loading,
      error,
      budget,
      recommendations,
      selectedTotal,
      isOverBudget,
      canPlaceOrder,
      placingOrder,
      orderError,
      successMessage,
      updateQuantity,
      placeOrder,
      formatCurrency,
      formatDate
    }
  }
}
</script>

<style scoped>
.budget-control {
  padding: 0.5rem 0;
}

.budget-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  margin-bottom: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-min,
.budget-max {
  font-size: 0.813rem;
  color: #64748b;
  flex-shrink: 0;
}

.budget-slider {
  flex: 1;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 110px;
  text-align: right;
  flex-shrink: 0;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
}

.summary-separator {
  color: #e2e8f0;
}

.summary-item.over-budget strong {
  color: #dc2626;
}

.budget-warning {
  background: #fecaca;
  color: #991b1b;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.qty-input {
  width: 80px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  text-align: right;
  outline: none;
  transition: border-color 0.15s;
}

.qty-input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.15);
}

.card-footer {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 1rem;
  margin-top: 0.5rem;
  border-top: 1px solid #e2e8f0;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #2563eb;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.order-error {
  color: #dc2626;
  font-size: 0.875rem;
}
</style>
