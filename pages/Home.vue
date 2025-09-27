<template>
  <v-app>
    <TheForm
      @updateData="handleUpdateData"
      @update:loading="isLoading = $event"
      @update:search="search = $event"
      @update:selectedPositions="selectedPositionsFromFilter = $event"
      @update:dates="handleDatesUpdate"
    />
    <v-btn 
      v-if="filteredItems.length"
      color="success"
      class="mt-4 ml-4"
      @click="sendTelegramNotification"
      :loading="isLoadingNotification"
      style="margin: 0.75rem 0 !important;"
    >
      Отправить в Telegram
    </v-btn>
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
          <template 
            v-for="position in filteredPositions" 
            :key="position.name" 
            class="text-center"
          >
            <!-- Итого заявки по должности -->
            <td class="v-data-table__td v-data-table-column--align-center">{{ positionApplicationTotals[position.name] || 0 }}</td>
            <!-- Итого выходы по должности -->
            <td class="v-data-table__td v-data-table-column--align-center">{{ positionExitTotals[position.name] || 0 }}</td>
          </template>
          <td class="v-data-table__td v-data-table-column--align-center">{{ allProjectsTotals.application }}</td>
          <td class="v-data-table__td v-data-table-column--align-center">{{ allProjectsTotals.exit }}</td>
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
const selectedPositionsFromFilter = ref([])
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
  },
  { 
    name: 'Пильщик', 
    applicationId: 'ufCrm35_1759007256', 
    exitId: 'ufCrm35_1759007284' 
  },
  { 
    name: 'Оператор', 
    applicationId: 'ufCrm35_1759007295', 
    exitId: 'ufCrm35_1759007307' 
  },
])

// Отфильтрованные должности на основе выбора в фильтре
const filteredPositions = computed(() => {
  if (!selectedPositionsFromFilter.value || selectedPositionsFromFilter.value.length === 0) {
    return positions.value;
  }
  
  return positions.value.filter(position => {
    return selectedPositionsFromFilter.value.some(selected => 
      selected.includes(position.applicationId) || selected.includes(position.exitId)
    );
  });
})

// Заголовки таблицы
const headers = computed(() => {
  const headerArray = [
    { text: 'Проект', value: 'projectName', align: 'start', sortable: true }
  ];
  
  // Добавляем заголовки для каждой выбранной должности (по два столбца: заявки и выходы)
  filteredPositions.value.forEach(position => {
    headerArray.push(
      {
        title: `${position.name} (Заявки)`,
        value: `${position.name}_application`,
        align: 'center',
        sortable: true
      },
      {
        title: `${position.name} (Выходы)`,
        value: `${position.name}_exit`,
        align: 'center',
        sortable: true
      }
    );
  });
  
  // Добавляем заголовки для итогов по проекту
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

// Обработанные данные для таблицы
const processedData = computed(() => {
  if (!leadsData.value.length) return [];
  
  const projectData = {};
  
  // Группируем данные по проектам
  leadsData.value.forEach(lead => {
    const projectId = lead.ufCrm35_Project;
    if (!projectId) return;
    
    if (!projectData[projectId]) {
      projectData[projectId] = {
        projectId: projectId,
        projectName: getProjectName(projectId),
        positions: {},
        applicationTotal: 0,
        exitTotal: 0,
      };
    }
    
    // Суммируем данные по выбранным должностям для этого проекта
    filteredPositions.value.forEach(position => {
      const applicationValue = getNumericValue(lead, position.applicationId);
      const exitValue = getNumericValue(lead, position.exitId);
      
      // Инициализируем объект для должности, если его нет
      if (!projectData[projectId].positions[position.name]) {
        projectData[projectId].positions[position.name] = {
          application: 0,
          exit: 0,
          total: 0
        };
      }
      
      // Добавляем значения заявок и выходов
      projectData[projectId].positions[position.name].application += applicationValue;
      projectData[projectId].positions[position.name].exit += exitValue;
      projectData[projectId].positions[position.name].total += (applicationValue + exitValue);
      
      // Добавляем к общим итогам проекта
      projectData[projectId].applicationTotal += applicationValue;
      projectData[projectId].exitTotal += exitValue;
    });
  });
  
  // Преобразуем в массив для таблицы
  return Object.values(projectData).map(project => {
    const rowData = {
      projectName: project.projectName,
      applicationTotal: project.applicationTotal,
      exitTotal: project.exitTotal,
    };
    
    // Добавляем данные по заявкам и выходам для каждой выбранной должности
    filteredPositions.value.forEach(position => {
      const positionData = project.positions[position.name] || { application: 0, exit: 0, total: 0 };
      rowData[`${position.name}_application`] = positionData.application;
      rowData[`${position.name}_exit`] = positionData.exit;
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
    if (item.projectName.toLowerCase().includes(searchTerm)) {
      return true;
    }
    
    // Поиск по значениям заявок и выходов
    for (const position of filteredPositions.value) {
      if (
        String(item[`${position.name}_application`] || '').toLowerCase().includes(searchTerm) ||
        String(item[`${position.name}_exit`] || '').toLowerCase().includes(searchTerm)
      ) {
        return true;
      }
    }
    
    // Поиск по итоговым значениям проекта
    if (
      String(item.applicationTotal).toLowerCase().includes(searchTerm) ||
      String(item.exitTotal).toLowerCase().includes(searchTerm)
    ) {
      return true;
    }
    
    return false;
  });
})

// Итоги по заявкам для каждой выбранной должности
const positionApplicationTotals = computed(() => {
  const totals = {};
  
  filteredPositions.value.forEach(position => {
    totals[position.name] = 0;
  });
  
  filteredItems.value.forEach(project => {
    filteredPositions.value.forEach(position => {
      totals[position.name] += project[`${position.name}_application`] || 0;
    });
  });
  
  return totals;
})

// Итоги по выходам для каждой выбранной должности
const positionExitTotals = computed(() => {
  const totals = {};
  
  filteredPositions.value.forEach(position => {
    totals[position.name] = 0;
  });
  
  filteredItems.value.forEach(project => {
    filteredPositions.value.forEach(position => {
      totals[position.name] += project[`${position.name}_exit`] || 0;
    });
  });
  
  return totals;
})

// Общие итоги по всем проектам
const allProjectsTotals = computed(() => {
  const totals = { application: 0, exit: 0, total: 0 };
  
  filteredItems.value.forEach(project => {
    totals.application += project.applicationTotal || 0;
    totals.exit += project.exitTotal || 0;
  });
  
  return totals;
})

// Данные для графика
const chartData = computed(() => {
  return {
    application: allProjectsTotals.value.application,
    exit: allProjectsTotals.value.exit,
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
const isLoadingNotification = ref(false)
const currentDates = ref([null, null]) // Храним текущие даты из фильтра
const handleDatesUpdate = (dates) => {
  currentDates.value = dates
  console.log('Получены даты:', dates)
}
// Функция для форматирования даты
const formatDate = (dateString) => {
  if (!dateString) return '';
  
  const date = new Date(dateString);
  return date.toLocaleDateString('ru-RU');
};

// Функция для получения заголовка с датами
const getDateTitle = () => {
  const [startDate, endDate] = currentDates.value;
  
  if (startDate && endDate) {
    return `Выход персонала на проекты за период: ${startDate} - ${endDate}`;
  } else if (startDate) {
    return `Выход персонала на проекты на дату: ${startDate}`;
  } else {
    return 'Выход персонала на проекты';
  }
};

// Функция для отправки уведомления в Telegram
const sendTelegramNotification = async () => {
  isLoadingNotification.value = true;
  
  try {
    // Формируем текст сообщения
    const message = formatTelegramMessage();
    
    // Настройки бота (замените на свои)
    const botToken = '7535274591:AAGpwGKwMkL8T3qJu508JwwJoFWrnDyNPOQ'; // Токен вашего бота
    const chatId = '-1002838872741'; // ID чата для отправки
    
    // Отправляем сообщение через Telegram Bot API
    const response = await fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        chat_id: chatId,
        text: message,
        parse_mode: 'HTML'
      })
    });
    
    const result = await response.json();
    
    if (result.ok) {
      console.log('Сообщение успешно отправлено в Telegram');
      // Можно добавить уведомление об успешной отправке
    } else {
      console.error('Ошибка отправки в Telegram:', result);
    }
  } catch (error) {
    console.error('Ошибка при отправке в Telegram:', error);
  } finally {
    isLoadingNotification.value = false;
  }
};

// Функция для форматирования сообщения для Telegram
const formatTelegramMessage = () => {
  const currentDate = new Date().toLocaleDateString('ru-RU');
  
  let message = `<b>${getDateTitle()}:</b>\n\n`;
  // Добавляем данные по каждому проекту
  filteredItems.value.forEach(project => {
    message += `<b>${project.projectName}</b>\n`;
    message += `Заявка клиента: ${project.applicationTotal}\n`;
    message += `Выход суточный: ${project.exitTotal}\n\n`;
  });
  
  // Добавляем итоговые значения
  message += `<b>ИТОГО</b>\n`;
  message += `Заявка клиентов: ${allProjectsTotals.value.application}\n`;
  message += `Выход суточный: ${allProjectsTotals.value.exit}`;
  
  return message;
};

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

/* Стили для лучшего отображения раздельных столбцов */
.v-data-table :deep(td) {
  padding: 8px 4px;
}

.v-data-table :deep(.text-center) {
  text-align: center;
}
</style>