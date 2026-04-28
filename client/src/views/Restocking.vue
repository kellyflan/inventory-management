<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-bounded restock recommendations from your demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submittedOrder" class="success-banner">
        <div class="success-content">
          <div class="success-title">Order {{ submittedOrder.order_number }} submitted</div>
          <div class="success-sub">
            Expected delivery {{ formatDate(submittedOrder.expected_delivery) }}.
            <router-link to="/orders">View in Orders</router-link>
          </div>
        </div>
        <button class="dismiss-btn" @click="submittedOrder = null">Dismiss</button>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <input
            type="range"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          >
          <div class="budget-values">
            <div class="budget-row">
              <span class="budget-label">Available budget</span>
              <span class="budget-value">${{ budget.toLocaleString() }}</span>
            </div>
            <div class="budget-row">
              <span class="budget-label">Selected total</span>
              <span class="budget-value">${{ selectedTotal.toLocaleString() }}</span>
            </div>
            <div class="budget-row" :class="overBudget ? 'warn' : 'ok'">
              <span class="budget-label">
                {{ overBudget ? 'Over budget by' : 'Remaining' }}
              </span>
              <span class="budget-value">
                ${{ Math.abs(budgetRemaining).toLocaleString() }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ candidates.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="!canPlaceOrder"
            @click="placeOrder"
          >
            Place Order ({{ selectedCount }} item{{ selectedCount === 1 ? '' : 's' }})
          </button>
        </div>

        <div v-if="candidates.length === 0" class="empty">
          No items currently need restocking — all forecasted demand is covered.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Forecast Gap</th>
                <th>Suggested Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
                <th class="col-include">Include</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="c in candidates"
                :key="c.sku"
                :class="{ 'row-selected': selection.has(c.sku) }"
              >
                <td><strong>{{ c.sku }}</strong></td>
                <td>{{ c.name }}</td>
                <td>+{{ c.gap }}</td>
                <td>{{ c.quantity }}</td>
                <td>${{ c.unit_cost.toFixed(2) }}</td>
                <td><strong>${{ c.line_total.toLocaleString() }}</strong></td>
                <td class="col-include">
                  <input
                    type="checkbox"
                    :checked="selection.has(c.sku)"
                    @change="toggleSelection(c.sku)"
                  >
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const allInventory = ref([])
    const budget = ref(50000)
    const selection = ref(new Set())
    const submittedOrder = ref(null)

    const inventoryBySku = computed(() => {
      const map = {}
      for (const item of allInventory.value) {
        map[item.sku] = item
      }
      return map
    })

    const candidates = computed(() => {
      return allForecasts.value
        .map(f => {
          const inv = inventoryBySku.value[f.item_sku]
          const gap = Math.max(0, f.forecasted_demand - f.current_demand)
          const unit_cost = inv?.unit_cost ?? 0
          return {
            sku: f.item_sku,
            name: f.item_name,
            gap,
            quantity: gap,
            unit_cost,
            line_total: gap * unit_cost,
          }
        })
        .filter(c => c.gap > 0 && c.unit_cost > 0)
        .sort((a, b) => b.gap - a.gap)
    })

    const selectedItems = computed(() =>
      candidates.value.filter(c => selection.value.has(c.sku))
    )
    const selectedTotal = computed(() =>
      selectedItems.value.reduce((s, c) => s + c.line_total, 0)
    )
    const selectedCount = computed(() => selectedItems.value.length)
    const budgetRemaining = computed(() => budget.value - selectedTotal.value)
    const overBudget = computed(() => selectedTotal.value > budget.value)
    const canPlaceOrder = computed(
      () => selectedCount.value > 0 && !overBudget.value
    )

    // Greedy fill: walk candidates (already sorted by gap desc) and include
    // each whose line_total still fits the budget. We skip over-budget items
    // instead of stopping so cheaper items lower in the list can still make it in.
    const greedyFill = () => {
      const next = new Set()
      let running = 0
      for (const c of candidates.value) {
        if (running + c.line_total <= budget.value) {
          next.add(c.sku)
          running += c.line_total
        }
      }
      selection.value = next
    }

    const toggleSelection = (sku) => {
      // Reassign a cloned Set so Vue picks up the change (Set internals aren't reactive).
      const next = new Set(selection.value)
      if (next.has(sku)) next.delete(sku)
      else next.add(sku)
      selection.value = next
    }

    const formatDate = (iso) => {
      return new Date(iso).toLocaleDateString('en-US', {
        year: 'numeric', month: 'short', day: 'numeric'
      })
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      const payload = {
        items: selectedItems.value.map(c => ({
          sku: c.sku,
          name: c.name,
          quantity: c.quantity,
          unit_price: c.unit_cost,
        })),
        budget_used: selectedTotal.value,
      }
      try {
        submittedOrder.value = await api.createRestockOrder(payload)
        selection.value = new Set()
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      }
    }

    const loadData = async () => {
      try {
        loading.value = true
        const [forecasts, inv] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory(),
        ])
        allForecasts.value = forecasts
        allInventory.value = inv
        greedyFill()
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch(budget, greedyFill)

    onMounted(loadData)

    return {
      loading, error, budget, submittedOrder,
      candidates, selection, selectedTotal,
      selectedCount, budgetRemaining, overBudget, canPlaceOrder,
      toggleSelection, placeOrder, formatDate,
    }
  }
}
</script>

<style scoped>
.budget-controls {
  display: grid;
  grid-template-columns: 1fr 280px;
  gap: 2rem;
  align-items: center;
  padding: 0.5rem 0;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 22px;
  height: 22px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 22px;
  height: 22px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-values {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
}

.budget-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.875rem;
}

.budget-row.ok {
  color: #059669;
  font-weight: 600;
}

.budget-row.warn {
  color: #dc2626;
  font-weight: 600;
}

.budget-label {
  color: #64748b;
}

.budget-row.ok .budget-label,
.budget-row.warn .budget-label {
  color: inherit;
}

.budget-value {
  font-variant-numeric: tabular-nums;
  color: #0f172a;
  font-weight: 600;
}

.budget-row.ok .budget-value,
.budget-row.warn .budget-value {
  color: inherit;
}

.col-include {
  width: 80px;
  text-align: center;
}

.row-selected {
  background: #eff6ff;
}

.row-selected:hover {
  background: #dbeafe !important;
}

input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #2563eb;
}

.place-order-btn {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.empty {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-left: 4px solid #059669;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-title {
  font-weight: 700;
  color: #065f46;
  font-size: 0.938rem;
}

.success-sub {
  color: #047857;
  font-size: 0.875rem;
  margin-top: 0.25rem;
}

.success-sub a {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
}

.dismiss-btn {
  background: transparent;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

@media (max-width: 768px) {
  .budget-controls {
    grid-template-columns: 1fr;
    gap: 1rem;
  }
}
</style>
