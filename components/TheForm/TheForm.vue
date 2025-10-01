<template lang="pug">
ErrorLog(
  v-model:dialogStages="dialogStages"
  :dialogMsg="dialogMsg"
)
v-form(v-model="formValid")
  Filtres(
    v-model:showInput="showInput"
    :users="users"
    :requestTypes="requestTypes"
    ref="child"
    @update:filters="updateFilters"
    @update:selectedPositions="emitSelectedPositions"
  )
  .container
    v-btn(color="primary" @click="submit" :loading="isLoading" style="height: 100%;") Получить заявки
    v-text-field(
        v-model="search"
        append-icon="mdi-magnify"
        label="Поиск"
        single-line
        hide-details
        variant="outlined"
        density="comfortable"
        @update:modelValue="emitSearch"
        clearable
    )
</template>

<script setup>
import { ref, defineEmits, defineProps, watch } from 'vue'
import { callApi } from '../../functions/callApi.ts'
import ErrorLog from './ErrorLog.vue'
import Filtres from './Filtres.vue'
import moment from 'moment'

const props = defineProps({
  monthNames: {
    type: Array
  },
  invoices: {
    type: Array
  },
  isLoading: {
    type: Boolean,
    default: false
  },
  showInput: {
    type: Array
  },
  parse: {
    type: Object
  },
  users: {
    type: Object
  },
  requestTypes: {
    type: Array,
    default: () => []
  }
})

const formValid = ref(true)
const dialogStages = ref(false)
const dialogMsg = ref(null)
const showInput = ref([false, false, false, false, false, false, false])
const child = ref(null)
const currentFilters = ref({})
const search = ref('');

// Добавляем emit для выбранных должностей
const emit = defineEmits(['update:search', 'updateData', 'update:loading', 'update:selectedPositions', 'update:dates'])

watch(search, (newSearch) => {
  emit('update:search', newSearch)
})

const updateFilters = (filters) => {
  currentFilters.value = filters
}

// Функция для передачи выбранных должностей в Home.vue
const emitSelectedPositions = (selectedPositions) => {
  emit('update:selectedPositions', selectedPositions)
}

const checkError = () => {
  console.log(currentFilters.value);
  if (showInput.value[5] && !currentFilters.value["<ufCrm35_Day"]) {
    dialogStages.value = true
    dialogMsg.value = 'Выберите дату'
    return true
  } else if (showInput.value[4]) {
    if (!currentFilters.value[">ufCrm35_Day"] || !currentFilters.value["<ufCrm35_Day"]) {
      dialogStages.value = true
      dialogMsg.value = 'Выберите дату'
      return true
    }
  }
  return false
}

const submit = async () => {
  if (child.value) {
    child.value.submit()
  }

  if (checkError()) {
    return
  }

  emit('update:loading', true)
  
  // Базовые поля, которые всегда нужны
  const baseFields = ["ufCrm35_Day", 'ufCrm35_Project'];
  
  // Получаем выбранные поля должностей из фильтров
  const selectedPositionFields = currentFilters.value.selectedPositionFields || [];
  
  // Объединяем базовые поля с выбранными полями должностей
  const selectFields = [...baseFields, ...selectedPositionFields];

  try {
    // Формируем фильтры на основе выбранных значений (исключаем selectedPositionFields)
    const { selectedPositionFields: _, ...filters } = currentFilters.value;
    
    console.log('Filters:', filters);
    console.log('Selected fields:', selectFields);
    
    const leads = await callApi("crm.item.list", 
      filters, 
      selectFields,
      1044
    )
    emit('update:dates', [moment(filters[">ufCrm35_Day"]).format('DD.MM.YYYY'), moment(filters["<ufCrm35_Day"]).format('DD.MM.YYYY')]);
    // Передаем данные напрямую родителю
    emit('updateData', leads);
    
  } catch (error) {
    console.error('Ошибка при получении данных:', error)
    dialogStages.value = true
    dialogMsg.value = 'Ошибка при получении данных'
  } finally {
    emit('update:loading', false)
  }
}

defineExpose({
  submit
})

</script>

<style lang="sass" scoped>
.v-form
  width: 100%
  display: flex
  flex-direction: column
  gap: 0.75rem
  margin-top: 0.5rem

.inputs
  display: grid
  grid-template-columns: repeat(auto-fit, minmax(20rem, 1fr))
  column-gap: 1rem
  padding: 0

.container
  display: grid
  grid-template-columns: 1fr 1fr
  gap: 0.75rem
</style>