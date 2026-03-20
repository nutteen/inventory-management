<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-display">${{ budget.toLocaleString() }}</span>
        </div>
        <div class="budget-slider-container">
          <input type="range" min="0" max="500000" step="1000" v-model.number="budget" class="budget-slider">
          <div class="budget-markers">
            <span>$0</span><span>$100K</span><span>$200K</span><span>$300K</span><span>$400K</span><span>$500K</span>
          </div>
        </div>
      </div>

      <!-- Stats Row -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemsSelected') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">${{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">${{ remainingBudget.toLocaleString() }}</div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.unitsToOrder') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><code>{{ item.sku }}</code></td>
                <td>{{ item.name }}</td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td>{{ item.unitsToOrder.toLocaleString() }}</td>
                <td>${{ item.unitCost.toLocaleString() }}</td>
                <td><strong>${{ item.totalCost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Action Bar -->
      <div class="action-bar">
        <button
          class="btn-primary"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder">
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>

      <!-- Success Banner (shown after order placed, stays visible) -->
      <div v-if="submitted" class="success-banner">
        {{ t('restocking.successMessage') }}
        <strong>{{ t('restocking.orderNumber') }}: {{ submittedOrderNumber }}</strong> —
        {{ t('restocking.expectedDelivery') }}: {{ submittedDeliveryDate }}
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
    const { t } = useI18n()

    const budget = ref(100000)
    const allDemandForecasts = ref([])
    const allOrders = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)
    const submittedOrderNumber = ref(null)
    const submittedDeliveryDate = ref(null)

    const unitCostMap = computed(() => {
      const map = {}
      for (const order of allOrders.value) {
        if (!order.items) continue
        for (const item of order.items) {
          if (item.sku && !(item.sku in map)) {
            map[item.sku] = item.unit_price
          }
        }
      }
      return map
    })

    const recommendations = computed(() => {
      const candidates = allDemandForecasts.value
        .filter(f => f.forecasted_demand > f.current_demand)
        .map(f => {
          const sku = f.item_sku
          const unitsToOrder = f.forecasted_demand - f.current_demand
          const unitCost = unitCostMap.value[sku] ?? 50
          const totalCost = unitsToOrder * unitCost
          return {
            sku,
            name: f.item_name,
            trend: f.trend,
            unitsToOrder,
            unitCost,
            totalCost
          }
        })

      // Sort: "increasing" trend first, then ascending totalCost within tier
      candidates.sort((a, b) => {
        const tierA = a.trend === 'increasing' ? 0 : 1
        const tierB = b.trend === 'increasing' ? 0 : 1
        if (tierA !== tierB) return tierA - tierB
        return a.totalCost - b.totalCost
      })

      // Greedy budget fill
      let remaining = budget.value
      const selected = []
      for (const item of candidates) {
        if (remaining >= item.totalCost) {
          selected.push(item)
          remaining -= item.totalCost
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, i) => sum + i.totalCost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, ordersData] = await Promise.all([
          api.getDemandForecasts(),
          api.getOrders({})
        ])
        allDemandForecasts.value = forecasts
        allOrders.value = ordersData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        const payload = {
          items: recommendations.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.unitsToOrder,
            unit_price: item.unitCost,
            total_cost: item.totalCost
          })),
          total_value: totalCost.value,
          budget: budget.value
        }
        const result = await api.submitRestockingOrder(payload)
        submittedOrderNumber.value = result.order_number
        const d = new Date(result.expected_delivery)
        submittedDeliveryDate.value = !isNaN(d.getTime())
          ? d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
          : result.expected_delivery
        submitted.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadData())

    return {
      t,
      budget,
      loading,
      error,
      submitting,
      submitted,
      submittedOrderNumber,
      submittedDeliveryDate,
      recommendations,
      totalCost,
      remainingBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider-container {
  padding: 0.75rem 0 0.5rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-markers {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.action-bar {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  margin-bottom: 1rem;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

code {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}
</style>
