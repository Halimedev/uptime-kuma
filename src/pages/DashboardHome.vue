<template>
    <transition ref="tableContainer" name="slide-fade" appear>
        <div v-if="$route.name === 'DashboardHome'">
            <h1 class="mb-3">
                {{ $t("Quick Stats") }}
            </h1>

            <div class="shadow-box big-padding text-center mb-4">
                <div class="row">
                    <div class="col">
                        <h3>{{ $t("Up") }}</h3>
                        <span class="num">{{ $root.stats.up }}</span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Down") }}</h3>
                        <span class="num text-danger">{{ $root.stats.down }}</span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Maintenance") }}</h3>
                        <span class="num text-maintenance">{{ $root.stats.maintenance }}</span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("Unknown") }}</h3>
                        <span class="num text-secondary">{{ $root.stats.unknown }}</span>
                    </div>
                    <div class="col">
                        <h3>{{ $t("pauseDashboardHome") }}</h3>
                        <span class="num text-secondary">{{ $root.stats.pause }}</span>
                    </div>
                </div>
            </div>

            <div class="filter-form mt-4 mx-auto" >
                <h1>Filtre des moniteurs par périodes</h1>
  <label>
    Début :
    <input type="date" v-model="customStartDate" class="form-control"/>
  </label>
  <label class="ms-3">
    Fin :
    <input type="date" v-model="customEndDate" class="form-control"/>
  </label>
  <label class="ms-3">
    Statuts :
    <multiselect
      v-model="selectedStatuses"
      :options="statusOptions"
      :multiple="true"
      :close-on-select="false"
      :clear-on-select="false"
      :preserve-search="true"
      placeholder="Sélectionnez un ou plusieurs statuts"
      label="name"
      track-by="value"
    />
  </label>
  <button class="btn btn-primary ms-3" @click="filterStatusByPeriod">
    Rechercher
  </button>
</div>
<div v-if="noResult" class="text-center text-danger mt-4">
  Aucun résultat trouvé pour les critères sélectionnés.
</div>

 
<table class="table table-bordered mt-4" v-if="paginatedFilteredHeartbeats.length">
  <thead>
    <tr>
      <th>Date/Heure</th>
      <th>Statut</th>
      <th>Message</th>
      <th>Monitor ID</th>
    </tr>
  </thead>
  <tbody>
    <tr v-for="hb in paginatedFilteredHeartbeats" :key="hb.id">
      <td>{{ hb.time }}</td>
      <td>
        <span v-if="hb.status === 0">🔴 Hors Ligne</span>
        <span v-else-if="hb.status === 1">🟢 En ligne</span>
        <span v-else-if="hb.status === 2">🟡 En attente</span>
        <span v-else-if="hb.status === 3">🟣 MAINTENANCE</span>
      </td>
      <td>{{ hb.msg }}</td>
      <td>{{ hb.monitor_id }}</td>
    </tr>
  </tbody>
</table>

<div class="d-flex justify-content-center kuma_pagination" v-if="filteredHeartbeats.length > perPageFiltered">
  <pagination
    v-model="pageFiltered"
    :records="filteredHeartbeats.length"
    :per-page="perPageFiltered"
    :options="{ hideCount: true, chunksNavigation: 'scroll' }"
  />
</div>


            <div class="shadow-box table-shadow-box" style="overflow-x: hidden;">
                <table class="table table-borderless table-hover">
                    <thead>
                        <tr>
                            <th>{{ $t("Name") }}</th>
                            <th>{{ $t("Status") }}</th>
                            <th>{{ $t("DateTime") }}</th>
                            <th>{{ $t("Message") }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(beat, index) in displayedRecords" :key="index" :class="{ 'shadow-box': $root.windowWidth <= 550 }">
                            <td><router-link :to="`/dashboard/${beat.monitorID}`">{{ beat.name }}</router-link></td>
                            <td><Status :status="beat.status" /></td>
                            <td :class="{ 'border-0': !beat.msg }"><Datetime :value="beat.time" /></td>
                            <td class="border-0">{{ beat.msg }}</td>
                        </tr>
                        <tr v-if="importantHeartBeatList.length === 0">
                            <td colspan="4">{{ $t("No important events") }}</td>
                        </tr>
                    </tbody>
                </table>


                <div class="d-flex justify-content-center kuma_pagination">
                    <pagination
                        v-model="page"
                        :records="importantHeartBeatList.length"
                        :per-page="perPage"
                        :options="paginationConfig"
                    />
                </div>
            </div>
        </div>
    </transition>
    <router-view ref="child" />
</template>

<script>
import Status from "../components/Status.vue";
import Datetime from "../components/Datetime.vue";
import Pagination from "v-pagination-3";
import Multiselect from "vue-multiselect";
import axios from "axios";


export default {



    components: {
        Datetime,
        Status,
        Pagination,
        Multiselect,
    },
    props: {
    calculatedHeight: {
        type: Number,
        default: 0
    }
},



    data() {
        return {
            pageFiltered: 1,
            perPageFiltered: 25,
            page: 1,
            perPage: 25,
            initialPerPage: 25,
            heartBeatList: [],
            customStartDate: "",
            customEndDate: "",
            selectedStatuses: [],
            statusOptions: [
                { name: "Hors ligne", value: 0 },
                { name: "En ligne", value: 1 },
                { name: "En attente", value: 2 },
                { name: "En Maintenance", value: 3 },
            ],
            filteredHeartbeats: [],
            noResult: false,
            paginationConfig: {
                hideCount: true,
                chunksNavigation: "scroll",
            },
        };
    },
    computed: {
        paginatedFilteredHeartbeats() {
    const start = (this.pageFiltered - 1) * this.perPageFiltered;
    const end = start + this.perPageFiltered;
    return this.filteredHeartbeats.slice(start, end);
  },
        importantHeartBeatList() {
            let result = [];
            for (let monitorID in this.$root.importantHeartbeatList) {
                let list = this.$root.importantHeartbeatList[monitorID];
                result = result.concat(list);
            }
            for (let beat of result) {
                let monitor = this.$root.monitorList[beat.monitorID];
                if (monitor) {
                    beat.name = monitor.name;
                }
            }
            result.sort((a, b) => b.time.localeCompare(a.time));
            this.heartBeatList = result;
            return result;
        },
        displayedRecords() {
            const startIndex = this.perPage * (this.page - 1);
            const endIndex = startIndex + this.perPage;
            return this.heartBeatList.slice(startIndex, endIndex);
        },
    },
    watch: {
        importantHeartBeatList() {
            this.$nextTick(() => {
                this.updatePerPage();
            });
        },
    },
    mounted() {
        this.initialPerPage = this.perPage;
        window.addEventListener("resize", this.updatePerPage);
        this.updatePerPage();
    },
    beforeUnmount() {
        window.removeEventListener("resize", this.updatePerPage);
    },
    methods: {
        updatePerPage() {
            const tableContainer = this.$refs.tableContainer;
            const tableContainerHeight = tableContainer.offsetHeight;
            const availableHeight = window.innerHeight - tableContainerHeight;
            const additionalPerPage = Math.floor(availableHeight / 58);
            this.perPage = additionalPerPage > 0 ? Math.max(this.initialPerPage, this.perPage + additionalPerPage) : this.initialPerPage;
        },
      async filterStatusByPeriod() {
    // Vérification des champs requis
    if (!this.customStartDate || !this.customEndDate) {
        alert("Veuillez sélectionner une date de début et une date de fin.");
        return;
    }

    const start = new Date(this.customStartDate);
    const end = new Date(this.customEndDate);

    if (end < start) {
        alert("La date de fin ne peut pas être antérieure à la date de début.");
        return;
    }

    if (this.selectedStatuses.length === 0) {
        alert("Veuillez sélectionner au moins un statut.");
        return;
    }

    try {
        const response = await axios.get("/api/status-by-period", {
            params: {
                start: this.customStartDate,
                end: this.customEndDate,
                statuses: this.selectedStatuses.map(s => s.value).join(",")
            }
        });

        if (response.data.success) {
            this.filteredHeartbeats = response.data.data;

            this.noResult = this.filteredHeartbeats.length === 0;
            console.log("Données récupérées :", this.filteredHeartbeats);
        } else {
            alert("Erreur API : " + (response.data.message || ""));
        }

    } catch (error) {
        console.error("Erreur API:", error);
        alert("Erreur lors du chargement des données.");
    }
}
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.num {
    font-size: 30px;
    color: $primary;
    font-weight: bold;
    display: block;
}

.shadow-box {
    padding: 20px;
}

.filter-form{
    text-align: center;
}

table {
    font-size: 14px;
    tr {
        transition: all ease-in-out 0.2ms;
    }
    @media (max-width: 550px) {
        table-layout: fixed;
        overflow-wrap: break-word;
    }
}
</style>
