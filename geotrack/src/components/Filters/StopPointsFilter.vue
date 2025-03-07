<template>
  <v-card
    class="mx-auto"
    width="100%"
    style="box-shadow: none; border-radius: 0 0 20px 20px; margin-bottom: 0px"
    color="primary"
  >
    <v-col style="padding: 5px 20px 0 20px">
      <v-row style="align-items: center; padding: 0px 0 7px 0; position: relative;">
        <!-- Texto "Pontos de Parada" centralizado -->
        <v-col cols="12" style="text-align: center;">
          <span style="font-size: 23px; font-weight: bold;" color="primary">
            Pontos de Parada
          </span>
        </v-col>
        <!-- Botão do Painel de Informações no extremo direito -->
        <v-col cols="auto" style="position: absolute; right: 0px; top: 32%; transform: translateY(-40%);">
          <InfoPanel></InfoPanel>
        </v-col>
      </v-row>
      <!-- Users selection -->
      <v-combobox
        v-model="selectedUsers"
        :items="users"
        label="Usuário"
        item-title="name"
        prepend-icon="mdi-filter-variant"
        chips
        clearable
        multiple
        color="secondary"
      >
        <template v-slot:selection="{ attrs, item, select, selected }">
          <v-chip
            v-bind="attrs"
            :model-value="selected"
            closable
            @click="select"
            @click:close="remove(item)"
          >
          </v-chip>
        </template>
      </v-combobox>

      <!-- Date selection -->
      <v-date-input
        v-model="date"
        label="Selecione o período"
        multiple="range"
        color="secondary"
        :max="today"
        :locale="locale"
        :format="customDateFormat"
        placeholder="dd/MM/yyyy"
        :readonly="dateInputDisabled"
        tooltip-data="Limite máximo de 30 dias"
      ></v-date-input>

      <!-- Quick date filters using chips -->
      <v-col style="padding: 0px; display: flex; justify-content: space-evenly">
        <v-chip
          v-for="(filter, index) in quickFilters"
          :key="filter.label"
          @click="setQuickFilter(filter.range, index)"
          :color="selectedQuickFilter === index ? 'primary' : 'primary_light'"
          :active="selectedQuickFilter === index"
          filter
          variant="flat"
          size="small"
        >
          {{ filter.label }}
        </v-chip>
      </v-col>
    </v-col>

    <v-card-actions class="d-flex" style="padding: 20px 20px 10px 20px">
      <v-row class="d-flex" no-gutters style="justify-content: space-around">
        <v-col cols="7">
          <v-btn
            :loading="loading"
            :disabled="ButtonDisabled || loading"
            class="text-none"
            color="secondary"
            size="large"
            variant="flat"
            block
            rounded="xl"
            @click="handleConsult"
          >
            Consultar
          </v-btn>
        </v-col>
        <v-col cols="4">
          <v-btn
            class="text-none"
            color="primary_light"
            size="large"
            variant="flat"
            block
            rounded="xl"
            @click="clearFields"
          >
            Limpar
          </v-btn>
        </v-col>
      </v-row>
    </v-card-actions>
  </v-card>

  <v-snackbar v-model="snackbar" :color="snackbarColor" timeout="3000" top>
    {{ snackbarMessage }}
  </v-snackbar>
</template>

<script>
import { eventBus } from "@/utils/EventBus";
import axios from "axios";
import InfoPanel from "../Info/InfoPanel.vue";

export default {
  data: () => ({
    today: new Date().toISOString().substr(0, 10),
    loading: false,
    logo: "/src/assets/Logo.svg",
    date: null,
    users: [], // Lista de usuários
    selectedUsers: [], // Armazena múltiplos usuários selecionados
    devices: [],
    locale: "pt",
    customDateFormat: "dd/MM/yyyy",
    quickFilters: [
      { label: "Hoje", range: [new Date(), new Date()] },
      {
        label: "Últimos 3 dias",
        range: [new Date(Date.now() - 3 * 864e5), new Date()],
      },
      {
        label: "Última semana",
        range: [new Date(Date.now() - 7 * 864e5), new Date()],
      },
      {
        label: "Último mês",
        range: [new Date(Date.now() - 30 * 864e5), new Date()],
      },
    ],
    selectedQuickFilter: null,
    dateInputDisabled: false,

    snackbar: false,
    snackbarColor: "success",
    snackbarMessage: "",
  }),

  emits: ["consult"],

  mounted() {
    this.fetchUsers();
    eventBus.on("stopIsLoading", this.stopIsLoading);
    eventBus.on("novoLogo", this.change);
    eventBus.on("clearFields", this.clearFields);
  },

  computed: {
    ButtonDisabled() {
      return this.selectedUsers.length === 0 || !this.date;
    },
  },

  methods: {
    change(newLogo) {
      this.logo = newLogo;
    },

    stopIsLoading() {
      this.loading = false;
    },

    showSnackbar(message, color = "success") {
      this.snackbarMessage = message; // Define a mensagem
      this.snackbarColor = color; // Define a cor
      this.snackbar = true; // Torna o snackbar visível
    },

    async fetchUsers() {
      try {
        const response = await axios.get(
          "http://localhost:8080/filters/users?page=0&qtdPage=1000"
        );
        const data = response.data;

        // Mapeando a resposta da API para o formato correto
        this.users = data.listUsers.map((user) => ({
          name: user.userName.toUpperCase(), // Nome do usuário
          deviceId: user.idDevice, // ID do dispositivo
        }));


      } catch (error) {

      }
    },

    async handleConsult() {
      if (this.selectedUsers.length === 0 || !this.date) return;

      this.loading = true;

      // Extraindo os IDs dos dispositivos dos usuários selecionados
      const deviceIds = this.selectedUsers.map((user) => user.deviceId);

      const qtddias = Math.round(
        (new Date(this.date[this.date.length - 1]) - new Date(this.date[0])) /
          (1000 * 60 * 60 * 24)
      );

      if (qtddias > 31) {
        this.showSnackbar("Mais que 31 dias selecionados", "error");
        this.loading = false;
        return;
      }

      // Preparando os dados da requisição com todos os devices como um array e as datas uma única vez
      const requestData = {
        devices: deviceIds, // Array de IDs dos dispositivos
        startDate: new Date(this.date[0]).toLocaleDateString("en-CA"), // Data de início
        finalDate: new Date(this.date[this.date.length - 1]).toLocaleDateString("en-CA"), // Data de fim
      };


      this.$emit("consult", requestData); // Certifique-se de emitir o evento com os dados

      // Simulação do retorno dos postos de parada
      setTimeout(() => {
        this.showStopPointsInformation = true; // Exibe o novo componente
        this.loadingPage = false;
      }, 2000); // Simulando o tempo de resposta
    },

    clearFields() {
      this.date = null;
      this.selectedUsers = [];
      this.devices = [];
      this.selectedQuickFilter = null;



      if (this.logo == "/src/assets/LogoWhite.svg") {
        this.$emit("initializeMapDark");
        eventBus.emit("clearStopPointsInformation");
      } else {
        this.$emit("initializeMap");
        eventBus.emit("clearStopPointsInformation");
      }
    },

    setQuickFilter(range, index) {
      if (this.selectedQuickFilter === index) {
        this.selectedQuickFilter = null;
        this.date = null;
        this.dateInputDisabled = false;
      } else {
        this.selectedQuickFilter = index;
        this.date = range.map((date) => date.toISOString().substr(0, 10));
        this.dateInputDisabled = true;
      }
    },

    remove(item) {
      this.selectedUsers = this.selectedUsers.filter(
        (user) => user.id !== item.id
      );
    },

    // Método para exibir o snackbar
    showSnackbar(message, color = "success") {
      this.snackbarMessage = message;
      this.snackbarColor = "error";
      this.snackbar = true;
    },

  },

  watch: {
    selectedUsers(newValue) {
      if (newValue.length > 5) {
        this.selectedUsers = newValue.slice(0, 5);
      }

    },
  },
};
</script>

<style scoped></style>
