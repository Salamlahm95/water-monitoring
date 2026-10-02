<script setup>
import { ref, computed, onMounted } from 'vue'
import { bitable } from '@lark-opdev/block-bitable-api'

const loading = ref(true)
const error = ref('')
const record = ref(null)
const fields = ref({})
const fieldList = ref([])
const allRecords = ref([])
const getFieldValue = (fieldId) => {
  const value = fields.value[fieldId]

  if (value == null) {
    return '-'
  }

  if (Array.isArray(value)) {
    return value
      .map(item => item.text ?? item.name ?? item.value ?? '')
      .filter(Boolean)
      .join(', ') || '-'
  }

  if (typeof value === 'object') {
    return JSON.stringify(value)
  }

  return String(value)
}

const getRecordValueByName = (record, name) => {
  const field = fieldList.value.find(f => f.name === name)

  if (!field) {
    return '-'
  }

  const value = record.fields?.[field.id]

  if (Array.isArray(value)) {
    return value
      .map(item => item.text ?? item.name ?? item.value ?? '')
      .filter(Boolean)
      .join(', ') || '-'
  }

  if (typeof value === 'object' && value !== null) {
    return value.text ?? value.name ?? value.value ?? '-'
  }

  return value ?? '-'
}

const cumulativeUsage = computed(() => {
  const dateField = fieldList.value.find(f => f.name === 'Date')
  const monthCycleField = fieldList.value.find(f => f.name === 'Month Cycle')
  const usageOpField = fieldList.value.find(f => f.name === 'Usage OP M')
  const usageFfField = fieldList.value.find(f => f.name === 'Usage FF M')

  if (!dateField || !monthCycleField || !usageOpField || !usageFfField) {
    return {
      op: 0,
      ff: 0,
      total: 0
    }
  }

  const currentDate = Number(fields.value[dateField.id])
  const currentMonthCycle = getValueByName('Month Cycle')

  let op = 0
  let ff = 0

  allRecords.value.forEach(item => {
    const itemDate = Number(item.fields?.[dateField.id])

    const itemMonthCycle =
      item.fields?.[monthCycleField.id]?.text ??
      ''

    if (
      itemMonthCycle === currentMonthCycle &&
      itemDate <= currentDate
    ) {
      op += Number(item.fields?.[usageOpField.id] || 0)
      ff += Number(item.fields?.[usageFfField.id] || 0)
    }
  })

  return {
    op,
    ff,
    total: op + ff
  }
})

const getValueByName = (name) => {
  const field = fieldList.value.find(f => f.name === name)

  if (!field) {
    return '-'
  }

  const value = fields.value[field.id]

if (Array.isArray(value)) {
  return value
    .map(item => item.text ?? item.name ?? item.value ?? '')
    .filter(Boolean)
    .join(', ') || '-'
}

if (typeof value === 'object' && value !== null) {
  return value.text ?? value.name ?? value.value ?? '-'
}

if (name === 'Over Average') {
  return Number(value) === 1 ? 'Yes' : 'No'
}

if (name === 'Date' && value) {
  const date = new Date(Number(value))

  const day = String(date.getDate()).padStart(2, '0')
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const year = date.getFullYear()

  return `${day}/${month}/${year}`
}

const meterFields = [
  'Operation Meter',
  'Usage OP M',
  'Average OP M',
  'FF Meter',
  'Usage FF M'
]

if (meterFields.includes(name) && value !== null && value !== undefined) {
  return `${value} m³`
}

const cartonFields = [
  'QHC FG',
  'CCI FG'
]

if (cartonFields.includes(name) && value !== null && value !== undefined) {
  return `${value} CTN`
}

return value ?? '-'
}

onMounted(async () => {
  try {
    const selection = await bitable.base.getSelection()

    console.log('Selection:', selection)

    if (!selection?.tableId || !selection?.recordId) {
      error.value = 'No record selected.'
      return
    }

    const table = await bitable.base.getTableById(selection.tableId)

    const result = await table.getRecordById(selection.recordId)

    console.log('Record:', result)

    record.value = result
    fields.value = result.fields || {}

    const response = await table.getRecords({
        pageSize: 200
    })

    console.log('ALL RECORDS:', response)

    allRecords.value = response.records

    const apiFields = await table.getFieldMetaList()

    console.log('API FIELDS:', apiFields)

    fieldList.value = apiFields.map(field => ({
        id: field.id,
        name: field.name
    }))
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Failed to load record.'
  } finally {
    loading.value = false
  }
})
</script>

<template>
    <main class="dashboard">
        <h1>Water Monitoring</h1>

        <p class="subtitle">
            Daily Water Usage Monitoring
        </p>

        

        <div class="period-info">
            <span>Date: </span>
            <strong>{{ getValueByName('Date') }}</strong>

            <span>Month Cycle: </span>
            <strong>{{ getValueByName('Month Cycle') }}</strong>
        </div>

        <div v-if="loading" class="message">
            Loading record...
        </div>

        <div v-else-if="error" class="message error">
            {{ error }}
        </div>

        <div v-else class="dashboard">

            <h3 class="section-title">Water Operation</h3>

            <div class="metrics-grid">

                <div class="metric-card">
                    <div class="metric-label">Operation Meter</div>
                    <div class="metric-value">
                        {{ getValueByName('Operation Meter') }}
                    </div>
                </div>

                <div class="metric-card">
                    <div class="metric-label">Water Usage - Operation</div>
                    <div class="metric-value">
                        {{ getValueByName('Usage OP M') }}
                    </div>
                </div>

                <div class="metric-card">
                    <div class="metric-label">Average Water Usage - Operation</div>
                    <div class="metric-value">
                        {{ getValueByName('Average OP M') }}
                    </div>
                </div>

                <div class="metric-card">
                    <div class="metric-label">Over Average</div>
                    <div class="status-badge"
                         :class="getValueByName('Over Average') === 'Yes' ? 'status-yes' : 'status-no'">
                        {{ getValueByName('Over Average') }}
                    </div>
                </div>

                <div class="metric-card cumulative-card cumulative-wide">
                    <div class="metric-label">Current Cumulative Water Usage - Operation</div>
                    <div class="metric-value">
                        {{ cumulativeUsage.op.toFixed(3) }} m³
                    </div>
                </div>

                <h3 class="section-title">Fire Fighting</h3>

                <div class="metric-card">
                    <div class="metric-label">Fire Fighting Meter</div>
                    <div class="metric-value">
                        {{ getValueByName('FF Meter') }}
                    </div>
                </div>

                <div class="metric-card">
                    <div class="metric-label">Water Usage - Fire Fighting</div>
                    <div class="metric-value">
                        {{ getValueByName('Usage FF M') }}
                    </div>
                </div>

                <div class="metric-card cumulative-card cumulative-wide">
                    <div class="metric-label">Current Cumulative Water Usage - Fire Fighting</div>
                    <div class="metric-value">
                        {{ cumulativeUsage.ff.toFixed(3) }} m³
                    </div>
                </div>

                <h3 class="section-title">Finished Goods</h3>

                <div class="metric-card">
                    <div class="metric-label">QHC FG</div>
                    <div class="metric-value">
                        {{ getValueByName('QHC FG') }}
                    </div>
                </div>

                <div class="metric-card">
                    <div class="metric-label">CCI FG</div>
                    <div class="metric-value">
                        {{ getValueByName('CCI FG') }}
                    </div>
                </div>

            </div>

        </div>

     </main>
</template>

<style scoped>
    .dashboard {
        padding: 16px;
    }

        .dashboard h1 {
            margin: 0 0 6px;
            font-size: 24px;
            font-weight: 700;
            color: #111827;
        }

    .subtitle {
        margin: 0 0 14px;
        font-size: 13px;
        color: #6b7280;
    }

    .period-info {
        display: flex;
        align-items: center;
        gap: 8px;
        margin-bottom: 20px;
        padding: 10px 12px;
        border: 1px solid #e5e7eb;
        border-radius: 8px;
        background: #f8fafc;
        font-size: 12px;
        color: #6b7280;
    }

        .period-info strong {
            color: #111827;
            margin-right: 12px;
        }

    .section-title {
        grid-column: 1 / -1;
        margin: 22px 0 8px;
        padding: 8px 10px;
        border-left: 4px solid #2563eb;
        border-bottom: 1px solid #e5e7eb;
        background: #D9F3FD;
        font-size: 14px;
        font-weight: 700;
        color: #1f2937;
        border-radius: 4px;
    }

    .metrics-grid {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 12px;
    }

    .metric-card {
        padding: 16px;
        border: 1px solid #e5e7eb;
        border-radius: 12px;
        background: #ffffff;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
        transition: transform 0.15s ease, box-shadow 0.15s ease;
    }

    .metric-card:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
    }

    .cumulative-card {
        border-left: 4px solid #2563eb;
        background: #f8fafc;
    }

    .cumulative-wide {
        grid-column: 1 / -1;
    }

    .cumulative-wide .metric-value {
        font-size: 24px;
        font-weight: 700;
    }
    .metric-label {
        font-size: 12px;
        color: #6b7280;
        margin-bottom: 6px;
    }

    .metric-value {
        font-size: 20px;
        font-weight: 600;
        color: #111827;
    }

    @media (max-width: 600px) {
        .metrics-grid {
            grid-template-columns: 1fr;
        }

        .period-info {
            flex-wrap: wrap;
            gap: 6px;
        }

            .period-info strong {
                margin-right: 8px;
            }

        .dashboard {
            padding: 12px;
        }

            .dashboard h1 {
                font-size: 20px;
            }

        .metric-card {
            padding: 14px;
        }

        .metric-value {
            font-size: 18px;
        }

        .cumulative-wide .metric-value {
            font-size: 22px;
        }
    }

    .status-badge {
        display: inline-flex;
        align-items: center;
        padding: 5px 12px;
        border-radius: 999px;
        font-size: 20px;
        font-weight: 700;
        letter-spacing: 0.2px;
    }

    .status-yes {
        background: #fee2e2;
        color: #b91c1c;
    }

    .status-no {
        background: #dcfce7;
        color: #15803d;
    }
</style>