<template>
    <div class="shadow-box mb-3" :style="boxStyle">
        <div class="list-header">
            <div class="header-top">
                <button class="btn btn-outline-normal ms-2" :class="{ 'active': selectMode }" type="button" @click="selectMode = !selectMode">
                    {{ $t("Select") }}
                </button>

                <div class="placeholder"></div>
                <div class="search-wrapper">
                    <a v-if="searchText == ''" class="search-icon">
                        <font-awesome-icon icon="search" />
                    </a>
                    <a v-if="searchText != ''" class="search-icon" @click="clearSearchText">
                        <font-awesome-icon icon="times" />
                    </a>
                    <form>
                        <input
                            v-model="searchText"
                            class="form-control search-input"
                            :placeholder="$t('Search...')"
                            :aria-label="$t('Search monitored sites')"
                            autocomplete="off"
                        />
                    </form>
                </div>
            </div>
            <div class="header-filter">
                <MonitorListFilter :filterState="filterState" @update-filter="updateFilter" />
            </div>

            <!-- Selection Controls -->
            <div v-if="selectMode" class="selection-controls px-2 pt-2">
                <input
                    v-model="selectAll"
                    class="form-check-input select-input"
                    type="checkbox"
                />

                <button class="btn-outline-normal" @click="pauseDialog"><font-awesome-icon icon="pause" size="sm" /> {{ $t("Pause") }}</button>
                <button class="btn-outline-normal" @click="resumeSelected"><font-awesome-icon icon="play" size="sm" /> {{ $t("Resume") }}</button>

                <span v-if="selectedMonitorCount > 0">
                    {{ $t("selectedMonitorCount", [ selectedMonitorCount ]) }}
                </span>
            </div>
        </div>

           <div class="filter-form mt-4">
  <label>
    Début :
    <input type="date" v-model="customStartDate" />
  </label>
  <label class="ms-3">
    Fin :
    <input type="date" v-model="customEndDate" />
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
 
<table class="table table-bordered mt-4" v-if="filteredHeartbeats.length">
  <thead>
    <tr>
      <th>Date/Heure</th>
      <th>Statut</th>
      <th>Message</th>
      <th>Monitor ID</th>
    </tr>
  </thead>
  <tbody>
    <tr v-for="hb in filteredHeartbeats" :key="hb.id">
      <td>{{ hb.time }}</td>
      <td>
        <span v-if="hb.status === 0">🔴 DOWN</span>
        <span v-else-if="hb.status === 1">🟢 UP</span>
        <span v-else-if="hb.status === 2">🟡 PENDING</span>
        <span v-else-if="hb.status === 3">🟣 MAINTENANCE</span>
      </td>
      <td>{{ hb.msg }}</td>
      <td>{{ hb.monitor_id }}</td>
    </tr>
  </tbody>
</table>

<PingChart
  v-if="filteredHeartbeats.length"
  :monitor-id="filteredHeartbeats[0].monitor_id"  
  :start-date="customStartDate"
  :end-date="customEndDate"
/>






        <div ref="monitorList" class="monitor-list" :class="{ scrollbar: scrollbar }" :style="monitorListStyle">
            <div v-if="Object.keys($root.monitorList).length === 0" class="text-center mt-3">
                {{ $t("No Monitors, please") }} <router-link to="/add">{{ $t("add one") }}</router-link>
            </div>

            <MonitorListItem
                v-for="(item, index) in sortedMonitorList"
                :key="index"
                :monitor="item"
                :isSelectMode="selectMode"
                :isSelected="isSelected"
                :select="select"
                :deselect="deselect"
                :filter-func="filterFunc"
                :sort-func="sortFunc"
            />
        </div>
    </div>
     






    <Confirm ref="confirmPause" :yes-text="$t('Yes')" :no-text="$t('No')" @yes="pauseSelected">
        {{ $t("pauseMonitorMsg") }}
    </Confirm>
</template>

<script>
import axios from "axios";
import Multiselect from "vue-multiselect";


import Confirm from "../components/Confirm.vue";
import MonitorListItem from "../components/MonitorListItem.vue";
import MonitorListFilter from "./MonitorListFilter.vue";
import { getMonitorRelativeURL } from "../util.ts";
import PingChart from "../components/PingChart.vue";


export default {
    components: {
        Confirm,
        MonitorListItem,
        MonitorListFilter,
        PingChart,
        Multiselect, 

    },
    props: {
        /** Should the scrollbar be shown */
        scrollbar: {
            type: Boolean,
        },
    },
    data() {
        return {
            searchText: "",
            selectMode: false,
            selectAll: false,
            disableSelectAllWatcher: false,
            selectedMonitors: {},
            customStartDate: "",
            customEndDate: "",
            selectedStatuses: [],
            selectedStatuses: [],
        statusOptions: [ // 👈 Ajout ici
            { name: "DOWN", value: 0 },
            { name: "UP", value: 1 },
            { name: "PENDING", value: 2 },
            { name: "MAINTENANCE", value: 3 },],
            filteredHeartbeats: [],
            windowTop: 0,
            filterState: {
                status: null,
                active: null,
                tags: null,
            }
        };
    },
    computed: {
        /**
         * Improve the sticky appearance of the list by increasing its
         * height as user scrolls down.
         * Not used on mobile.
         * @returns {object} Style for monitor list
         */
        boxStyle() {
            if (window.innerWidth > 550) {
                return {
                    height: `calc(100vh - 160px + ${this.windowTop}px)`
                };
            } else {
                return {
                    height: "calc(100vh - 160px)"
                };
            }

        },

        /**
         * Returns a sorted list of monitors based on the applied filters and search text.
         * @returns {Array} The sorted list of monitors.
         */
        sortedMonitorList() {
            let result = Object.values(this.$root.monitorList);

            result = result.filter(monitor => {
                // The root list does not show children
                if (monitor.parent !== null) {
                    return false;
                }
                return true;
            });

            result = result.filter(this.filterFunc);

            result.sort(this.sortFunc);

            return result;
        },

        isDarkTheme() {
            return document.body.classList.contains("dark");
        },

        monitorListStyle() {
            let listHeaderHeight = 107;

            if (this.selectMode) {
                listHeaderHeight += 42;
            }

            return {
                "height": `calc(100% - ${listHeaderHeight}px)`
            };
        },

        selectedMonitorCount() {
            return Object.keys(this.selectedMonitors).length;
        },

        /**
         * Determines if any filters are active.
         * @returns {boolean} True if any filter is active, false otherwise.
         */
        filtersActive() {
            return this.filterState.status != null || this.filterState.active != null || this.filterState.tags != null || this.searchText !== "";
        },
    },
    watch: {
        searchText() {
            for (let monitor of this.sortedMonitorList) {
                if (!this.selectedMonitors[monitor.id]) {
                    if (this.selectAll) {
                        this.disableSelectAllWatcher = true;
                        this.selectAll = false;
                    }
                    break;
                }
            }
        },
        selectAll() {
            if (!this.disableSelectAllWatcher) {
                this.selectedMonitors = {};

                if (this.selectAll) {
                    this.sortedMonitorList.forEach((item) => {
                        this.selectedMonitors[item.id] = true;
                    });
                }
            } else {
                this.disableSelectAllWatcher = false;
            }
        },
        selectMode() {
            if (!this.selectMode) {
                this.selectAll = false;
                this.selectedMonitors = {};
            }
        },
    },
    mounted() {
        window.addEventListener("scroll", this.onScroll);
    },
    beforeUnmount() {
        window.removeEventListener("scroll", this.onScroll);
    },
    methods: {









        /** Handle user scroll */
        onScroll() {
            if (window.top.scrollY <= 133) {
                this.windowTop = window.top.scrollY;
            } else {
                this.windowTop = 133;
            }
        },
        /**
         * Get URL of monitor
         * @param {number} id ID of monitor
         * @returns {string} Relative URL of monitor
         */
        monitorURL(id) {
            return getMonitorRelativeURL(id);
        },
        /**
         * Clear the search bar
         * @returns {void}
         */
        clearSearchText() {
            this.searchText = "";
        },
        /**
         * Update the MonitorList Filter
         * @param {object} newFilter Object with new filter
         * @returns {void}
         */
        updateFilter(newFilter) {
            this.filterState = newFilter;
        },
        /**
         * Deselect a monitor
         * @param {number} id ID of monitor
         * @returns {void}
         */
        deselect(id) {
            delete this.selectedMonitors[id];
        },
        /**
         * Select a monitor
         * @param {number} id ID of monitor
         * @returns {void}
         */
        select(id) {
            this.selectedMonitors[id] = true;
        },
        /**
         * Determine if monitor is selected
         * @param {number} id ID of monitor
         * @returns {bool} Is the monitor selected?
         */
        isSelected(id) {
            return id in this.selectedMonitors;
        },
        /**
         * Disable select mode and reset selection
         * @returns {void}
         */
        cancelSelectMode() {
            this.selectMode = false;
            this.selectedMonitors = {};
        },
        /**
         * Show dialog to confirm pause
         * @returns {void}
         */
        pauseDialog() {
            this.$refs.confirmPause.show();
        },
        /**
         * Pause each selected monitor
         * @returns {void}
         */
        pauseSelected() {
            Object.keys(this.selectedMonitors)
                .filter(id => this.$root.monitorList[id].active)
                .forEach(id => this.$root.getSocket().emit("pauseMonitor", id, () => {}));

            this.cancelSelectMode();
        },
        /**
         * Resume each selected monitor
         * @returns {void}
         */
        resumeSelected() {
            Object.keys(this.selectedMonitors)
                .filter(id => !this.$root.monitorList[id].active)
                .forEach(id => this.$root.getSocket().emit("resumeMonitor", id, () => {}));

            this.cancelSelectMode();
        },




        async filterStatusByPeriod() {
  if (!this.customStartDate || !this.customEndDate || this.selectedStatuses.length === 0) {
    alert("Veuillez remplir tous les champs.");
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
      console.log("Données récupérées :", this.filteredHeartbeats);
    } else {
      alert("Erreur API : " + (response.data.message || ""));
    }

  } catch (error) {
    console.error("Erreur API:", error);
    alert("Erreur lors du chargement des données");
  }
}

  }
        



    
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";
@import "vue-multiselect/dist/vue-multiselect.css";


.shadow-box {
    height: calc(100vh - 150px);
    position: sticky;
    top: 10px;
}

.small-padding {
    padding-left: 5px !important;
    padding-right: 5px !important;
}

.list-header {
    border-bottom: 1px solid #dee2e6;
    border-radius: 10px 10px 0 0;
    margin: -10px;
    margin-bottom: 10px;
    padding: 10px;

    .dark & {
        background-color: $dark-header-bg;
        border-bottom: 0;
    }
}

.header-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.header-filter {
    display: flex;
    align-items: center;
}

@media (max-width: 770px) {
    .list-header {
        margin: -20px;
        margin-bottom: 10px;
        padding: 5px;
    }
}

.search-wrapper {
    display: flex;
    align-items: center;
}

.search-icon {
    padding: 10px;
    color: #c0c0c0;

    // Clear filter button (X)
    svg[data-icon="times"] {
        cursor: pointer;
        transition: all ease-in-out 0.1s;

        &:hover {
            opacity: 0.5;
        }
    }
}

.search-input {
    max-width: 15em;
}

.monitor-item {
    width: 100%;
}

.tags {
    margin-top: 4px;
    padding-left: 67px;
    display: flex;
    flex-wrap: wrap;
    gap: 0;
}

.bottom-style {
    padding-left: 67px;
    margin-top: 5px;
}

.selection-controls {
    margin-top: 5px;
    display: flex;
    align-items: center;
    gap: 10px;
}
</style>
