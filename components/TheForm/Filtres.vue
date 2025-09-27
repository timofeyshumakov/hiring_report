<template lang="pug">
.container
  Date(
    :showInput="showInput"
    :selectedDateIso="selectedDateIso"
    @update:date="updateDate"
    @update:dateRange="updateDateRange"
  )
  v-autocomplete(
        v-model="selectedProjects"
        :items="projects"
        item-title="name"
        item-value="id"
        label="Проект"
        multiple
        chips
        @update:modelValue="emitFilters"
        variant="outlined"
        clearable
  )
        template(v-slot:prepend-item)
          v-list-item
            v-list-item-content
              v-list-item-title
                v-checkbox(
                  label="Выбрать все проекты" 
                  :modelValue="selectedAllProjects" 
                  @change="toggleAllProjects()"
                )
  v-autocomplete(
        v-model="selectedPositions"
        :items="positions"
        item-title="name"
        item-value="id"
        label="Должность"
        multiple
        chips
        @update:modelValue="emitFilters"
        variant="outlined"
        clearable
  )
        template(v-slot:prepend-item)
          v-list-item
            v-list-item-content
              v-list-item-title
                v-checkbox(
                  label="Выбрать все должности" 
                  :modelValue="selectedAllPositions" 
                  @change="toggleAllPositions()"
                )
  v-autocomplete(
        v-model="selectedResponsibles"
        :items="responsibles"
        item-title="name"
        item-value="id"
        label="Ответственный"
        multiple
        chips
        @update:modelValue="emitFilters"
        variant="outlined"
        clearable
  )
        template(v-slot:prepend-item)
          v-list-item
            v-list-item-content
              v-list-item-title
                v-checkbox(
                  label="Выбрать всех ответственных" 
                  :modelValue="selectedAllResponsibles" 
                  @change="toggleAllResponsibles()"
                )
</template>

<script setup>
import { ref, defineProps, defineEmits, computed, onMounted } from 'vue'
import Date from './Date/Date.vue'

const projects = ref([
  { id: 9619, name: "ВИ ДМД" },
  { id: 9621, name: "НАМ-НЯМ" },
  { id: 9623, name: "АГАМА" },
  { id: 9625, name: "СПЛАТ" },
  { id: 9627, name: "ТАБЛОДЖИКС" }
]);

const positions = ref([
  { 
    id: 'ufCrm35_1757591746_ufCrm35_1757591875',
    name: 'Комплектовщик (СПК)', 
    applicationId: 'ufCrm35_1757591746', 
    exitId: 'ufCrm35_1757591875' 
  },
  { 
    id: 'ufCrm35_1757591893_ufCrm35_1757591850',
    name: 'СПК ППО', 
    applicationId: 'ufCrm35_1757591893', 
    exitId: 'ufCrm35_1757591850' 
  },
  { 
    id: 'ufCrm35_1757591910_ufCrm35_1757592055',
    name: 'Грузчик', 
    applicationId: 'ufCrm35_1757591910', 
    exitId: 'ufCrm35_1757592055' 
  },
  { 
    id: 'ufCrm35_1757591920_ufCrm35_1757592066',
    name: 'Упаковщик', 
    applicationId: 'ufCrm35_1757591920', 
    exitId: 'ufCrm35_1757592066' 
  },
  { 
    id: 'ufCrm35_1757591931_ufCrm35_1757592077',
    name: 'Разнорабочий', 
    applicationId: 'ufCrm35_1757591931', 
    exitId: 'ufCrm35_1757592077' 
  },
  { 
    id: 'ufCrm35_1757591943_ufCrm35_1757592085',
    name: 'Повар', 
    applicationId: 'ufCrm35_1757591943', 
    exitId: 'ufCrm35_1757592085' 
  },
  { 
    id: 'ufCrm35_1757591951_ufCrm35_1757592094',
    name: 'Помощник повара', 
    applicationId: 'ufCrm35_1757591951', 
    exitId: 'ufCrm35_1757592094' 
  },
  { 
    id: 'ufCrm35_1757591964_ufCrm35_1757592104',
    name: 'Котломойщик', 
    applicationId: 'ufCrm35_1757591964', 
    exitId: 'ufCrm35_1757592104' 
  },
  { 
    id: 'ufCrm35_1757591974_ufCrm35_1757592112',
    name: 'Уборщик', 
    applicationId: 'ufCrm35_1757591974', 
    exitId: 'ufCrm35_1757592112' 
  }
  ,
  { 
    id: 'ufCrm35_1759007256_ufCrm35_1759007284',
    name: 'Пильщик', 
    applicationId: 'ufCrm35_1759007256', 
    exitId: 'ufCrm35_1759007284' 
  },
  { 
    id: 'ufCrm35_1759007295_ufCrm35_1759007307',
    name: 'Оператор', 
    applicationId: 'ufCrm35_1759007295', 
    exitId: 'ufCrm35_1759007307' 
  }
])

const responsibles = ref([
  { id: 42067, name: "Рыгзынова Тамара" },
  { id: 778, name: "Торган Марина" },
  { id: 5243, name: "Алонсо Суарес Анна" },
]);

const selectedProjects = ref([]);
const selectedPositions = ref([]);
const selectedResponsibles = ref([]);

const props = defineProps({
  showInput: Array,
})

// Добавляем emit для выбранных должностей
const emit = defineEmits(['update:filters', 'update:selectedPositions'])

const selectedDateIso = ref([null, null])

// Вычисляемое свойство для определения, выбраны ли все проекты
const selectedAllProjects = computed(() => {
  return selectedProjects.value.length === projects.value.length
})

// Вычисляемое свойство для определения, выбраны ли все должности
const selectedAllPositions = computed(() => {
  return selectedPositions.value.length === positions.value.length
})

// Вычисляемое свойство для определения, выбраны ли все ответственные
const selectedAllResponsibles = computed(() => {
  return selectedResponsibles.value.length === responsibles.value.length
})

// Функция для выбора/снятия всех проектов
const toggleAllProjects = () => {
  if (selectedAllProjects.value) {
    selectedProjects.value = []
  } else {
    selectedProjects.value = projects.value.map(project => project.id)
  }
  emitFilters()
}

// Функция для выбора/снятия всех должностей
const toggleAllPositions = () => {
  if (selectedAllPositions.value) {
    selectedPositions.value = []
  } else {
    selectedPositions.value = positions.value.map(position => position.id)
  }
  emitFilters()
}

// Функция для выбора/снятия всех ответственных
const toggleAllResponsibles = () => {
  if (selectedAllResponsibles.value) {
    selectedResponsibles.value = []
  } else {
    selectedResponsibles.value = responsibles.value.map(responsible => responsible.id)
  }
  emitFilters()
}

const updateDate = (date) => {
  selectedDateIso.value = [date, null]
  emitFilters()
}

const updateDateRange = (range) => {
  selectedDateIso.value = range
  emitFilters()
}

// Функция для получения ID полей выбранных должностей
const getSelectedPositionFields = () => {
  const selectedFields = [];
  
  selectedPositions.value.forEach(positionId => {
    const position = positions.value.find(p => p.id === positionId);
    if (position) {
      selectedFields.push(position.applicationId);
      selectedFields.push(position.exitId);
    }
  });
  
  return selectedFields;
}

// Функция для получения выбранных должностей (для передачи в родительский компонент)
const getSelectedPositions = () => {
  const selected = [];
  
  selectedPositions.value.forEach(positionId => {
    const position = positions.value.find(p => p.id === positionId);
    if (position) {
      selected.push(position.applicationId);
      selected.push(position.exitId);
    }
  });
  
  return selected;
}

const emitFilters = () => {
  const filters = {}
  
  // Добавляем фильтры по дате
  if (selectedDateIso.value[0]) {
    filters['>ufCrm35_Day'] = selectedDateIso.value[0]
  }
  if (selectedDateIso.value[1]) {
    filters['<ufCrm35_Day'] = selectedDateIso.value[1]
  }
  
  // Добавляем фильтры по проектам (ID)
  if (selectedProjects.value.length > 0) {
    filters['ufCrm35_Project'] = selectedProjects.value
  }
  
  // Добавляем фильтры по ответственным (ID)
  if (selectedResponsibles.value.length > 0) {
    filters['assignedById'] = selectedResponsibles.value
  }
  
  // Передаем выбранные поля должностей в отдельном свойстве
  const selectedPositionFields = getSelectedPositionFields();
  
  // Передаем выбранные должности для родительского компонента
  const selectedPositionsForParent = getSelectedPositions();
  
  emit('update:filters', {
    ...filters,
    selectedPositionFields
  })
  
  // Отправляем выбранные должности в родительский компонент
  emit('update:selectedPositions', selectedPositionsForParent)
}

// Инициализация - выбираем все проекты, должности и ответственных по умолчанию
onMounted(() => {
  selectedProjects.value = projects.value.map(project => project.id)
  selectedPositions.value = positions.value.map(position => position.id)
  selectedResponsibles.value = responsibles.value.map(responsible => responsible.id)
  emitFilters()
})

// Метод для вызова из родителя
const submit = () => {
  emitFilters()
}

defineExpose({
  submit
})
</script>

<style lang="sass">
.v-field__field
  max-height: 7rem
  overflow: hidden

.container
  display: grid
  grid-template-columns: 1fr 1fr
</style>