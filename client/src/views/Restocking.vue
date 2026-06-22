<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error && !successOrder" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="successOrder" class="success-banner">
        <span>
          Order <strong>{{ successOrder.order_number }}</strong> placed successfully.
          Expected delivery: {{ formatDate(successOrder.expected_delivery) }}
        </span>
        <a href="#" class="reset-link" @click.prevent="successOrder = null">Place Another Order</a>
      </div>

      <!-- Budget card -->
      <div class="card budget-card">
        <div class="budget-label">{{ t('restocking.budgetSlider') }}</div>
        <div class="budget-amount">{{ formatCurrency(budget) }}</div>
        <input
          type="range"
          min="0"
          max="120000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="budget-meta-row">
          <span>{{ t('restocking.budgetUsed') }}: {{ formatCurrency(totalCost) }}</span>
          <span>{{ t('restocking.budgetRemaining') }}: {{ formatCurrency(budget - totalCost) }}</span>
        </div>
      </div>

      <!-- Recommended Restocking Order card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedOrder') }}</h3>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('inventory.table.sku') }}</th>
                <th>{{ t('inventory.table.itemName') }}</th>
                <th>{{ t('restocking.trend') }}</th>
                <th>{{ t('demand.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.unitCost') }}</th>
                <th>{{ t('restocking.total') }}</th>
                <th></th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedForecasts"
                :key="item.item_sku"
                :class="{ excluded: !selectedSkus.has(item.item_sku) }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.forecasted_demand * item.unit_cost) }}</strong></td>
                <td class="excluded-label">
                  <span v-if="!selectedSkus.has(item.item_sku)" class="excluded-text">Excluded</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="order-footer">
          <span class="footer-stat">{{ t('restocking.budgetUsed') }}: {{ formatCurrency(totalCost) }}</span>
          <span class="footer-stat">{{ selectedItems.length }} items selected</span>
          <button
            class="place-order-btn"
            :disabled="!selectedItems.length || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, currentLocale } = useI18n()

    const forecasts = ref([])
    const budget = ref(50000)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successOrder = ref(null)

    const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

    const sortedForecasts = computed(() =>
      [...forecasts.value].sort((a, b) =>
        TREND_PRIORITY[a.trend] - TREND_PRIORITY[b.trend]
      )
    )

    // Greedy: include items while budget allows
    const selectedItems = computed(() => {
      let remaining = budget.value
      return sortedForecasts.value.filter(item => {
        const cost = item.forecasted_demand * item.unit_cost
        if (cost <= remaining) { remaining -= cost; return true }
        return false
      })
    })

    const selectedSkus = computed(() => new Set(selectedItems.value.map(i => i.item_sku)))

    const totalCost = computed(() =>
      selectedItems.value.reduce((sum, i) => sum + i.forecasted_demand * i.unit_cost, 0)
    )

    const formatCurrency = (value) => {
      const symbol = currentCurrency.value === 'JPY' ? '¥' : '$'
      return symbol + Math.round(value).toLocaleString()
    }

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!selectedItems.value.length || submitting.value) return
      submitting.value = true
      error.value = null
      try {
        const order = await api.submitRestockingOrder(selectedItems.value)
        successOrder.value = order
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadForecasts())

    return {
      t,
      forecasts,
      budget,
      loading,
      error,
      submitting,
      successOrder,
      sortedForecasts,
      selectedItems,
      selectedSkus,
      totalCost,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

/* Budget card */
.budget-card {
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.25rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
  margin-bottom: 0.75rem;
}

.budget-meta-row {
  display: flex;
  gap: 2rem;
  font-size: 0.875rem;
  color: #64748b;
}

/* Restocking table */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.restocking-table th:nth-child(1) { width: 110px; }
.restocking-table th:nth-child(2) { width: auto; }
.restocking-table th:nth-child(3) { width: 110px; }
.restocking-table th:nth-child(4) { width: 130px; }
.restocking-table th:nth-child(5) { width: 110px; }
.restocking-table th:nth-child(6) { width: 110px; }
.restocking-table th:nth-child(7) { width: 90px; }

/* Excluded row styling */
.excluded {
  opacity: 0.4;
}

.excluded-label {
  text-align: right;
}

.excluded-text {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  padding: 2px 8px;
}

/* Order footer */
.order-footer {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 10px 10px;
}

.footer-stat {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.place-order-btn {
  margin-left: auto;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f0fdf4;
  border: 1px solid #86efac;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.5rem;
  font-size: 0.9rem;
  color: #166534;
}

.reset-link {
  color: #2563eb;
  font-weight: 600;
  font-size: 0.875rem;
  text-decoration: none;
  white-space: nowrap;
  margin-left: 1.5rem;
}

.reset-link:hover {
  text-decoration: underline;
}
</style>
