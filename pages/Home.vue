<template>
  <v-app>
    <TheForm
      @updateData="handleUpdateData"
      @update:loading="isLoading = $event"
      @update:search="search = $event"
      @update:selectedPositions="selectedPositionsFromFilter = $event"
    />
    
    <!-- Таблица с данными по проектам -->
    <v-data-table
      v-show="processedData.length"
      :headers="headers"
      :items="filteredItems"
      class="elevation-1 mt-4"
      :loading="isLoading"
    >
      <template v-slot:body.append>
        <tr class="font-weight-bold">
          <td>Итого по должностям</td>
          <td 
            v-for="position in filteredPositions" 
            :key="position.name" 
            class="text-center"
          >
            {{ positionTotals[position.name] || 0 }}
          </td>
          <td class="text-center">{{ allProjectsTotals.application }}</td>
          <td class="text-center">{{ allProjectsTotals.exit }}</td>
          <td class="text-center">{{ allProjectsTotals.total }}</td>
        </tr>
      </template>
    </v-data-table>
    
    <!-- График Highcharts -->
    <div v-show="filteredItems.length" class="chart-container mt-6">
      <div id="chart" style="width: 100%; height: 400px;"></div>
    </div>
  </v-app>
</template>

<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
import TheForm from '../components/TheForm/TheForm.vue'
import Highcharts from 'highcharts'

const users = ref({})
const requestTypes = ref([])
const isLoading = ref(false)
const search = ref('')
const leadsData = ref([])
const selectedPositionsFromFilter = ref([]) // Выбранные должности из фильтра
let chart = null

const projects = ref([
  { id: 9619, name: "ВИ ДМД" },
  { id: 9621, name: "НАМ-НЯМ" },
  { id: 9623, name: "АГАМА" },
  { id: 9625, name: "СПЛАТ" },
  { id: 9627, name: "ТАБЛОДЖИКС" }
]);

const positions = ref([
  { 
    name: 'Комплектовщик (СПК)', 
    applicationId: 'ufCrm35_1757591746', 
    exitId: 'ufCrm35_1757591875' 
  },
  { 
    name: 'СПК ППО', 
    applicationId: 'ufCrm35_1757591893', 
    exitId: 'ufCrm35_1757591850' 
  },
  { 
    name: 'Грузчик', 
    applicationId: 'ufCrm35_1757591910', 
    exitId: 'ufCrm35_1757592055' 
  },
  { 
    name: 'Упаковщик', 
    applicationId: 'ufCrm35_1757591920', 
    exitId: 'ufCrm35_1757592066' 
  },
  { 
    name: 'Разнорабочий', 
    applicationId: 'ufCrm35_1757591931', 
    exitId: 'ufCrm35_1757592077' 
  },
  { 
    name: 'Повар', 
    applicationId: 'ufCrm35_1757591943', 
    exitId: 'ufCrm35_1757592085' 
  },
  { 
    name: 'Помощник повара', 
    applicationId: 'ufCrm35_1757591951', 
    exitId: 'ufCrm35_1757592094' 
  },
  { 
    name: 'Котломойщик', 
    applicationId: 'ufCrm35_1757591964', 
    exitId: 'ufCrm35_1757592104' 
  },
  { 
    name: 'Уборщик', 
    applicationId: 'ufCrm35_1757591974', 
    exitId: 'ufCrm35_1757592112' 
  }
])

// Отфильтрованные должности на основе выбора в фильтре
const filteredPositions = computed(() => {
  if (!selectedPositionsFromFilter.value || selectedPositionsFromFilter.value.length === 0) {
    return positions.value; // Если ничего не выбрано, показываем все
  }
  
  return positions.value.filter(position => {
    // Проверяем, есть ли должность в выбранных фильтрах
    return selectedPositionsFromFilter.value.some(selected => 
      selected.includes(position.applicationId) || selected.includes(position.exitId)
    );
  });
})

// Заголовки таблицы (проекты как строки, только выбранные должности как столбцы)
const headers = computed(() => {
  const headerArray = [
    { text: 'Проект', value: 'projectName', align: 'start', sortable: true }
  ];
  
  // Добавляем заголовки только для выбранных должностей
  filteredPositions.value.forEach(position => {
    headerArray.push({
      title: position.name,
      value: position.name,
      align: 'center',
      sortable: true
    });
  });
  
  // Добавляем заголовки для заявки, выхода и итого по проекту
  headerArray.push(
    {
      title: 'Заявка по проекту',
      value: 'applicationTotal',
      align: 'center',
      sortable: true
    },
    {
      title: 'Выход по проекту',
      value: 'exitTotal',
      align: 'center',
      sortable: true
    },
    {
      title: 'Итого по проекту',
      value: 'projectTotal',
      align: 'center',
      sortable: true
    }
  );
  
  return headerArray;
})

const handleUpdateData = (leads) => {
  console.log('Получены обновленные данные:', leads)
  leadsData.value = leads;
}

// Функция для получения числового значения из поля сделки
const getNumericValue = (lead, fieldId) => {
  const value = lead[fieldId];
  if (value === undefined || value === null) return 0;
  const num = Number(value);
  return isNaN(num) ? 0 : num;
}

// Функция для получения названия проекта по ID
const getProjectName = (projectId) => {
  const project = projects.value.find(p => p.id === parseInt(projectId));
  return project ? project.name : `Проект ${projectId}`;
}

// Обработанные данные для таблицы (проекты как строки)
const processedData = computed(() => {
  if (!leadsData.value.length) return [];
  
  const projectData = {};
  
  // Группируем данные по проектам
  leadsData.value.forEach(lead => {
    const projectId = lead.ufCrm35_Project; // ID проекта из поля сделки
    if (!projectId) return;
    
    if (!projectData[projectId]) {
      projectData[projectId] = {
        projectId: projectId,
        projectName: getProjectName(projectId),
        positions: {},
        applicationTotal: 0,
        exitTotal: 0,
        projectTotal: 0
      };
    }
    
    // Суммируем данные только по выбранным должностям для этого проекта
    filteredPositions.value.forEach(position => {
      const applicationValue = getNumericValue(lead, position.applicationId);
      const exitValue = getNumericValue(lead, position.exitId);
      const totalValue = applicationValue + exitValue;
      
      if (!projectData[projectId].positions[position.name]) {
        projectData[projectId].positions[position.name] = 0;
      }
      
      projectData[projectId].positions[position.name] += totalValue;
      projectData[projectId].applicationTotal += applicationValue;
      projectData[projectId].exitTotal += exitValue;
      projectData[projectId].projectTotal += totalValue;
    });
  });
  
  // Преобразуем в массив для таблицы
  return Object.values(projectData).map(project => {
    const rowData = {
      projectName: project.projectName,
      applicationTotal: project.applicationTotal,
      exitTotal: project.exitTotal,
      projectTotal: project.projectTotal
    };
    
    // Добавляем данные только по выбранным должностям
    filteredPositions.value.forEach(position => {
      rowData[position.name] = project.positions[position.name] || 0;
    });
    
    return rowData;
  });
})

// Элементы для таблицы с фильтрацией по поиску
const filteredItems = computed(() => {
  if (!search.value) {
    return processedData.value;
  }
  
  const searchTerm = search.value.toLowerCase();
  return processedData.value.filter(item => {
    // Поиск по названию проекта
    if (item.projectName.toLowerCase().includes(searchTerm)) {
      return true;
    }
    
    // Поиск по значениям в выбранных должностях
    for (const position of filteredPositions.value) {
      if (String(item[position.name] || '').toLowerCase().includes(searchTerm)) {
        return true;
      }
    }
    
    // Поиск по итоговым значениям проекта
    if (
      String(item.applicationTotal).toLowerCase().includes(searchTerm) ||
      String(item.exitTotal).toLowerCase().includes(searchTerm) ||
      String(item.projectTotal).toLowerCase().includes(searchTerm)
    ) {
      return true;
    }
    
    return false;
  });
})

// Итоги по каждой выбранной должности (для строки итогов)
const positionTotals = computed(() => {
  const totals = {};
  
  // Инициализируем итоги для каждой выбранной должности
  filteredPositions.value.forEach(position => {
    totals[position.name] = 0;
  });
  
  // Суммируем данные по всем проектам (учитывая фильтрацию)
  filteredItems.value.forEach(project => {
    filteredPositions.value.forEach(position => {
      totals[position.name] += project[position.name] || 0;
    });
  });
  
  return totals;
})

// Общие итоги по всем проектам (учитывая фильтрацию)
const allProjectsTotals = computed(() => {
  const totals = { application: 0, exit: 0, total: 0 };
  
  filteredItems.value.forEach(project => {
    totals.application += project.applicationTotal || 0;
    totals.exit += project.exitTotal || 0;
    totals.total += project.projectTotal || 0;
  });
  
  return totals;
})

// Данные для графика
const chartData = computed(() => {
  return {
    application: allProjectsTotals.value.application,
    exit: allProjectsTotals.value.exit,
    total: allProjectsTotals.value.total
  };
})

// Функция для создания графика
const createChart = () => {
  if (chart) {
    chart.destroy();
  }
  
  chart = Highcharts.chart('chart', {
    chart: {
      type: 'column'
    },
    title: {
      text: 'Итоговые значения по заявке и выходу',
      align: 'center'
    },
    subtitle: {
      text: 'Общая статистика по всем проектам',
      align: 'center'
    },
    xAxis: {
      categories: ['Заявка', 'Выход', 'Итого'],
      title: {
        text: null
      },
      gridLineWidth: 1,
      lineWidth: 0
    },
    yAxis: {
      min: 0,
      title: {
        text: '',
        align: 'high'
      },
      labels: {
        overflow: 'justify'
      },
      gridLineWidth: 0
    },
    tooltip: {
      valueSuffix: ''
    },
    plotOptions: {
      column: {
        borderRadius: 5,
        dataLabels: {
          enabled: true,
          format: '{y}'
        }
      }
    },
    legend: {
      enabled: false
    },
    credits: {
      enabled: false
    },
    series: [{
      name: '',
      data: [
        {
          name: 'Заявка',
          y: chartData.value.application,
          color: '#4CAF50'
        },
        {
          name: 'Выход',
          y: chartData.value.exit,
          color: '#2196F3'
        },
      ]
    }]
  });
}

// Следим за изменениями данных для обновления графика
watch(chartData, (newData) => {
  if (newData.application > 0 || newData.exit > 0) {
    createChart();
  }
}, { deep: true });

// Следим за изменениями поиска для обновления графика
watch(search, () => {
  if (chartData.value.application > 0 || chartData.value.exit > 0) {
    createChart();
  }
});

// Следим за изменениями выбранных должностей
watch(selectedPositionsFromFilter, () => {
  // При изменении фильтра должностей пересчитываем данные
  if (chartData.value.application > 0 || chartData.value.exit > 0) {
    createChart();
  }
});

onMounted(() => {
  // Init users, requestTypes from API if needed
});

onUnmounted(() => {
  if (chart) {
    chart.destroy();
  }
});
</script>

<style scoped>
.chart-container {
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
</style>