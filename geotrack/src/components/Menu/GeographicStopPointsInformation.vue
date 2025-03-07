<template>
  <v-card class="stop-points-card" width="400px" height="300px" rounded="xl" elevation="4" color="primary">
    <template v-if="geoStopPoints && Object.keys(geoStopPoints).length > 0">
      <v-list class="scrollable-container bg-primary">
        <v-list-item>
          <v-sheet class="pa-2 mb-2" color="primary_light" elevation="0" rounded>
            <v-row align="center" justify="space-between" style="padding: 5px 12px;">
              <span style="font-weight: bold; font-size: 16px;">
                {{ getUserInitials(geoStopPoints.userName) }}
              </span>
              <span outlined>
                {{ geoStopPoints.device.toUpperCase() }}
              </span>
            </v-row>
          </v-sheet>
          <v-list-item v-for="(coord, coordIndex) in geoStopPoints.coords" :key="coordIndex"
            @click="navigateToStopPoint(coord)" style="padding: 0;" color="primary">
            <v-row align="start" no-gutters class="px-2 py-1">
              <v-col cols="1">
                <v-icon color="secondary" class="mr-1 mt-1">mdi-map-marker</v-icon>
              </v-col>
              <v-col cols="11">
                <v-list-item-title>
                  {{ displayedAddresses[coordIndex] }}
                </v-list-item-title>
                <v-list-item-subtitle>
                  {{ formatDateRange(coord.startDate, coord.endDate) }}
                </v-list-item-subtitle>
                <v-list-item-subtitle>
                  Tempo parado: {{ calculateStopDuration(coord.startDate, coord.endDate) }}
                </v-list-item-subtitle>
              </v-col>
            </v-row>
            <v-divider v-if="coordIndex < geoStopPoints.coords.length"></v-divider>
          </v-list-item>
        </v-list-item>
      </v-list>
    </template>
    <template v-else>
      <v-row justify="center" align="center">
        <v-icon color="grey">mdi-alert</v-icon>
        <span>Nenhum ponto de parada encontrado.</span>
      </v-row>
    </template>
  </v-card>
</template>

<script setup lang="ts">
import { defineEmits, defineProps, ref, watch } from 'vue';

const props = defineProps<{
  geoStopPoints: {
    userName: string;
    device: string;
    coords: Array<{
      latitude: number;
      longitude: number;
      startDate: string;
      endDate: string;
    }>;
  }
}>();

const emit = defineEmits<{
  (e: 'navigate-to-stop-point', coordinates: [number, number]): void;
}>();

const addresses = ref<string[][]>([]);
const displayedAddresses = ref<string[][]>([]);

const getUserInitials = (fullName: string) => {
  const names = fullName.split(' ');
  return names.slice(0, 2).join(' '); // Retorna os dois primeiros nomes
};

const getFormattedAddress = async (lat: number, lng: number) => {
  const apiKey = 'AIzaSyD7OMuCxvuTi0AfzyZLoChSWkjUzNQj8Lk';
  const url = `https://maps.googleapis.com/maps/api/geocode/json?latlng=${lat},${lng}&key=${apiKey}&language=BR&region`;

  try {
    const response = await fetch(url);
    const data = await response.json();


    const formattedAddress = data.results.length > 0 ? data.results[0].formatted_address : 'Endereço não encontrado';

    return formattedAddress;
  } catch (error) {

    return 'Erro ao buscar endereço';
  }
};

const fetchAddresses = async () => {
  addresses.value = [];
  displayedAddresses.value = [];
  for (const coord of props.geoStopPoints.coords) {
    const address = await getFormattedAddress(coord.latitude, coord.longitude);
    addresses.value.push([address]); // Inicializa cada endereço como array de strings
    displayedAddresses.value.push(address); // Exibe inicialmente apenas o primeiro endereço
  }
};

const formatDateRange = (startDate: string, endDate: string) => {
  const parseDate = (date: string) => new Date(date);
  const formatDatePart = (date: Date) =>
    date.toLocaleDateString('pt-BR', { day: '2-digit', month: '2-digit', year: 'numeric' });
  const formatTimePart = (date: Date) =>
    date.toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' });

  const start = parseDate(startDate);
  const end = parseDate(endDate);

  return `${formatDatePart(start)}, ${formatTimePart(start)} - ${formatTimePart(end)}`;
};

const calculateStopDuration = (startDate: string, endDate: string) => {
  const start = new Date(startDate).getTime();
  const end = new Date(endDate).getTime();
  const durationMinutes = Math.floor((end - start) / (1000 * 60));

  const hours = Math.floor(durationMinutes / 60);
  const minutes = durationMinutes % 60;

  return `${hours} hora${hours !== 1 ? 's' : ''} e ${minutes} minuto${minutes !== 1 ? 's' : ''}`;
};

const navigateToStopPoint = (coord: { latitude: number; longitude: number }) => {

  emit('navigate-to-stop-point', [coord.latitude, coord.longitude]);
};

watch(
  () => props.geoStopPoints,
  fetchAddresses,
  { immediate: true }
);
</script>

<style scoped>
.stop-points-card {
  box-shadow: none;
  border-radius: 0;
  position: fixed;
  margin-top: 15px;
}

.scrollable-container {
  max-height: 300px;
  /* Ajuste a altura conforme necessário */
  overflow-y: auto;
  overflow-x: hidden;
}

.scrollable-container::-webkit-scrollbar {
  width: 10px;
}

.scrollable-container::-webkit-scrollbar-track {
  background: #f0f0f5;
  border-radius: 5px;
}

.scrollable-container::-webkit-scrollbar-thumb {
  background: linear-gradient(180deg, #c7c9ff, #7a7de9);
  border-radius: 10px;
}

.scrollable-container::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(180deg, #6b6fe1, #4f51d3);
}

.v-list-item-title {
  word-wrap: break-word;
  white-space: normal;
}
</style>
