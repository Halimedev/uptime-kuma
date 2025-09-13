<template>
    <div>
        <div class="period-options">
            <button type="button" class="btn btn-light dropdown-toggle btn-period-toggle" data-bs-toggle="dropdown" aria-expanded="false">
                {{ chartPeriodOptions[chartPeriodHrs] }}&nbsp;
            </button>
            <ul class="dropdown-menu dropdown-menu-end">
                <li v-for="(item, key) in chartPeriodOptions" :key="key">
                    <a class="dropdown-item" :class="{ active: chartPeriodHrs == key }" href="#" @click="chartPeriodHrs = key">{{ item }}</a>
                </li>
            </ul>
        </div>

              <div v-if="chartPeriodHrs === 'custom'" class="filter-form mt-4">
  <label>
    
    <input type="date" v-model="customStartDate" class="form-control"/>
  </label>
  <label class="ms-3">
    
    <input type="date" v-model="customEndDate" class="form-control"/>
  </label>
  
  <button class="btn btn-primary ms-3" @click="filtergraphByPeriod">
    Rechercher
  </button>
</div>


        <div class="chart-wrapper" :class="{ loading : loading}">
            <Line :data="chartData" :options="chartOptions" />
        </div>
    </div>
    


</template>

<script lang="js">
import { BarController, BarElement, Chart, Filler, LinearScale, LineController, LineElement, PointElement, TimeScale, Tooltip } from "chart.js";
import "chartjs-adapter-dayjs-4";
import dayjs from "dayjs";
import { Line } from "vue-chartjs";
import { useToast } from "vue-toastification";
import { DOWN, PENDING, MAINTENANCE, log } from "../util.ts";

const toast = useToast();

Chart.register(LineController, BarController, LineElement, PointElement, TimeScale, BarElement, LinearScale, Tooltip, Filler);

export default {
    components: { Line },
    props: {
        /** ID of monitor */
        monitorId: {
            type: Number,
            required: true,
        },

        startDate: {
        type: String,
        required: false
    },
    endDate: {
        type: String,
        required: false
    },
    },
    data() {
        return {

            loading: false,

            // Configurable filtering on top of the returned data
            chartPeriodHrs: 0,

            chartPeriodOptions: {
                0: this.$t("recent"),
                3: "3h",
                6: "6h",
                24: "24h",
                168: "1w",
                custom:"Personnaliser"
            },

            // A heartbeatList for 3h, 6h, 24h, 1w
            // Uses the $root.heartbeatList when value is null
            heartbeatList: null,

            customStartDate: "",
            customEndDate: "",
        };
    },
    computed: {
        chartOptions() {
            return {
                responsive: true,
                maintainAspectRatio: false,
                onResize: (chart) => {
                    chart.canvas.parentNode.style.position = "relative";
                    if (screen.width < 576) {
                        chart.canvas.parentNode.style.height = "275px";
                    } else if (screen.width < 768) {
                        chart.canvas.parentNode.style.height = "320px";
                    } else if (screen.width < 992) {
                        chart.canvas.parentNode.style.height = "300px";
                    } else {
                        chart.canvas.parentNode.style.height = "250px";
                    }
                },
                layout: {
                    padding: {
                        left: 10,
                        right: 30,
                        top: 30,
                        bottom: 10,
                    },
                },

                elements: {
                    point: {
                        // Hide points on chart unless mouse-over
                        radius: 0,
                        hitRadius: 100,
                    },
                },
                scales: {
                    x: {
                        type: "time",
                        time: {
                            minUnit: "minute",
                            round: "second",
                            tooltipFormat: "YYYY-MM-DD HH:mm:ss",
                            displayFormats: {
                                minute: "HH:mm",
                                hour: "MM-DD HH:mm",
                            }
                        },
                        ticks: {
                            sampleSize: 3,
                            maxRotation: 0,
                            autoSkipPadding: 30,
                            padding: 3,
                        },
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                            offset: false,
                        },
                    },
                    y: {
                        title: {
                            display: true,
                            text: this.$t("respTime"),
                        },
                        offset: false,
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                        },
                    },
                    y1: {
                        display: false,
                        position: "right",
                        grid: {
                            drawOnChartArea: false,
                        },
                        min: 0,
                        max: 1,
                        offset: false,
                    },
                },
                bounds: "ticks",
                plugins: {
                    tooltip: {
                        mode: "nearest",
                        intersect: false,
                        padding: 10,
                        backgroundColor: this.$root.theme === "light" ? "rgba(212,232,222,1.0)" : "rgba(32,42,38,1.0)",
                        bodyColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        titleColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        filter: function (tooltipItem) {
                            return tooltipItem.datasetIndex === 0;  // Hide tooltip on Bar Chart
                        },
                        callbacks: {
                            label: (context) => {
                                return ` ${new Intl.NumberFormat().format(context.parsed.y)} ms`;
                            },
                        }
                    },
                    legend: {
                        display: false,
                    },
                },
            };
        },
        chartData() {
            let pingData = [];  // Ping Data for Line Chart, y-axis contains ping time
            let downData = [];  // Down Data for Bar Chart, y-axis is 1 if target is down (red color), under maintenance (blue color) or pending (orange color), 0 if target is up
            let colorData = []; // Color Data for Bar Chart

            let heartbeatList = this.heartbeatList ||
             (this.monitorId in this.$root.heartbeatList && this.$root.heartbeatList[this.monitorId]) ||
             [];

            heartbeatList
                                .filter((beat) => {
                    const beatTime = dayjs.utc(beat.time).tz(this.$root.timezone);

                    const afterStart = this.startDate ? beatTime.isAfter(dayjs(this.startDate).startOf("day")) : true;
                    const beforeEnd = this.endDate ? beatTime.isBefore(dayjs(this.endDate).endOf("day")) : true;

                    return afterStart && beforeEnd;
                })

                .map((beat) => {
                    const x = this.$root.datetime(beat.time);
                    pingData.push({
                        x,
                        y: beat.ping,
                    });
                    downData.push({
                        x,
                        y: (beat.status === DOWN || beat.status === MAINTENANCE || beat.status === PENDING) ? 1 : 0,
                    });
                    colorData.push((beat.status === MAINTENANCE) ? "rgba(23,71,245,0.41)" : ((beat.status === PENDING) ? "rgba(245,182,23,0.41)" : "#DC354568"));
                });

            return {
                datasets: [
                    {
                        // Line Chart
                        data: pingData,
                        fill: "origin",
                        tension: 0.2,
                        borderColor: "#5CDD8B",
                        backgroundColor: "#5CDD8B38",
                        yAxisID: "y",
                        label: "ping",
                    },
                    {
                        // Bar Chart
                        type: "bar",
                        data: downData,
                        borderColor: "#00000000",
                        backgroundColor: colorData,
                        yAxisID: "y1",
                        barThickness: "flex",
                        barPercentage: 1,
                        categoryPercentage: 1,
                        inflateAmount: 0.05,
                        label: "status",
                    },
                ],
            };
        },
    },


    methods: {
    async filtergraphByPeriod() {
  // Vérification des champs obligatoires
  if (!this.customStartDate || !this.customEndDate) {
    toast.error("Veuillez sélectionner une date de début et une date de fin.");
    return;
  }

  const start = new Date(this.customStartDate);
  const end = new Date(this.customEndDate);

  // Vérification que la date de fin n'est pas antérieure à la date de début
  if (end < start) {
    toast.error("La date de fin ne peut pas être antérieure à la date de début.");
    return;
  }

  // Optionnel : limiter la période à 1 an maximum pour éviter surcharge
  const maxPeriodDays = 365;
  const diffTime = end.getTime() - start.getTime();
  const diffDays = diffTime / (1000 * 3600 * 24);
  if (diffDays > maxPeriodDays) {
    toast.error(`La période ne peut pas dépasser ${maxPeriodDays} jours.`);
    return;
  }

  this.loading = true;

  try {
    const url = `http://localhost:3001/api/status-by-period?start=${this.customStartDate}&end=${this.customEndDate}&statuses=1&monitorId=${this.monitorId}`;
    const res = await fetch(url);

    // Vérification du type de réponse
    const contentType = res.headers.get("content-type");
    if (!res.ok) {
      const errorText = await res.text();
      console.error("Erreur serveur :", errorText);
      toast.error("Le serveur a renvoyé une erreur.");
      this.loading = false;
      return;
    }
    if (!contentType || !contentType.includes("application/json")) {
      const errorText = await res.text();
      console.error("Réponse non-JSON :", errorText);
      toast.error("Le serveur a renvoyé une réponse invalide.");
      this.loading = false;
      return;
    }

    const json = await res.json();

    if (!json.success) {
      toast.error(json.message || "Erreur lors de la récupération des données.");
    } else {
      this.heartbeatList = json.data;
      this.chartPeriodHrs = 0; 
    }
  } catch (e) {
    toast.error("Erreur réseau. Veuillez vérifier votre connexion.");
    console.error("Erreur dans filtergraphByPeriod:", e);
  } finally {
    this.loading = false;
  }

  this.$emit("update-custom-dates", {
            start: this.customStartDate,
            end: this.customEndDate
        });
}

},




    watch: {
        // Update chart data when the selected chart period changes
        chartPeriodHrs: function (newPeriod) {

            // eslint-disable-next-line eqeqeq
            if (newPeriod == "0") {
                this.heartbeatList = null;
                this.$root.storage().removeItem(`chart-period-${this.monitorId}`);
            } else {
                this.loading = true;

                this.$root.getMonitorBeats(this.monitorId, newPeriod, (res) => {
                    if (!res.ok) {
                        toast.error(res.msg);
                    } else {
                        this.heartbeatList = res.data;
                        this.$root.storage()[`chart-period-${this.monitorId}`] = newPeriod;
                    }
                    this.loading = false;
                });
            }
        }
    },
    created() {
        // Setup Watcher on the root heartbeatList,
        // And mirror latest change to this.heartbeatList
        this.$watch(() => this.$root.heartbeatList[this.monitorId],
            (heartbeatList) => {

                log.debug("ping_chart", `this.chartPeriodHrs type ${typeof this.chartPeriodHrs}, value: ${this.chartPeriodHrs}`);

                // eslint-disable-next-line eqeqeq
                if (this.chartPeriodHrs != "0") {
                    const newBeat = heartbeatList.at(-1);
                    if (newBeat && dayjs.utc(newBeat.time) > dayjs.utc(this.heartbeatList.at(-1)?.time)) {
                        this.heartbeatList.push(heartbeatList.at(-1));
                    }
                }
            },
            { deep: true }
        );

        // Load chart period from storage if saved
        let period = this.$root.storage()[`chart-period-${this.monitorId}`];
        if (period != null) {
            this.chartPeriodHrs = Math.min(period, 6);
        }
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.form-select {
    width: unset;
    display: inline-flex;
}

.period-options {
    padding: 0.1em 1em;
    margin-bottom: -1.2em;
    float: right;
    position: relative;
    z-index: 10;

    .dropdown-menu {
        padding: 0;
        min-width: 50px;
        font-size: 0.9em;

        .dark & {
            background: $dark-bg;
        }

        .dropdown-item {
            border-radius: 0.3rem;
            padding: 2px 16px 4px;

            .dark & {
                background: $dark-bg;
            }

            .dark &:hover {
                background: $dark-font-color;
                color: $dark-font-color2;
            }
        }

        .dark & .dropdown-item.active {
            background: $primary;
            color: $dark-font-color2;
        }
    }

    .btn-period-toggle {
        padding: 2px 15px;
        background: transparent;
        border: 0;
        color: $link-color;
        opacity: 0.7;
        font-size: 0.9em;

        &::after {
            vertical-align: 0.155em;
        }

        .dark & {
            color: $dark-font-color;
        }
    }
}

.chart-wrapper {
    margin-bottom: 0.5em;

    &.loading {
        filter: blur(10px);
    }
}
</style>
