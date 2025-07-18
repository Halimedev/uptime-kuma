<template>
    <div>
        <div class="period-options">
            <button
                type="button" class="btn btn-light dropdown-toggle btn-period-toggle" data-bs-toggle="dropdown"
                aria-expanded="false"
            >
                {{ chartPeriodOptions[chartPeriodHrs] }}&nbsp;
            </button>
            <ul class="dropdown-menu dropdown-menu-end">
                <li v-for="(item, key) in chartPeriodOptions" :key="key">
                    <button
                        type="button" class="dropdown-item" :class="{ active: chartPeriodHrs == key }"
                        @click="chartPeriodHrs = key"
                    >
                        {{ item }}
                    </button>
                </li>
            </ul>
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
import { Line } from "vue-chartjs";
import { UP, DOWN, PENDING, MAINTENANCE } from "../util.ts";

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

            // Time period for the chart to display, in hours
            // Initial value is 0 as a workaround for triggering a data fetch on created()
            chartPeriodHrs: "0",

            chartPeriodOptions: {
                0: this.$t("recent"),
                3: "3h",
                6: "6h",
                24: "24h",
                168: "1w",
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
            if (this.chartPeriodHrs === "0") {
                return this.getChartDatapointsFromHeartbeatList();
            } else {
                return this.getChartDatapointsFromStats();
            }
        },
    },
    watch: {
        // Update chart data when the selected chart period changes
        chartPeriodHrs: function (newPeriod) {
            if (this.chartDataFetchInterval) {
                clearInterval(this.chartDataFetchInterval);
                this.chartDataFetchInterval = null;
            }

            // eslint-disable-next-line eqeqeq
            if (newPeriod == "0") {
                this.heartbeatList = null;
                this.$root.storage().removeItem(`chart-period-${this.monitorId}`);
            } else {
                this.loading = true;

                let period;
                try {
                    period = parseInt(newPeriod);
                } catch (e) {
                    // Invalid period
                    period = 24;
                }

                this.$root.getMonitorChartData(this.monitorId, period, (res) => {
                    if (!res.ok) {
                        this.$root.toastError(res.msg);
                    } else {
                        this.chartRawData = res.data;
                        this.$root.storage()[`chart-period-${this.monitorId}`] = newPeriod;
                    }
                    this.loading = false;
                });

                this.chartDataFetchInterval = setInterval(() => {
                    this.$root.getMonitorChartData(this.monitorId, period, (res) => {
                        if (res.ok) {
                            this.chartRawData = res.data;
                        }
                    });
                }, 5 * 60 * 1000);
            }
        }
    },
    created() {
        // Load chart period from storage if saved
        let period = this.$root.storage()[`chart-period-${this.monitorId}`];
        if (period != null) {
            // Has this ever been not a string?
            if (typeof period !== "string") {
                period = period.toString();
            }
            this.chartPeriodHrs = period;
        } else {
            this.chartPeriodHrs = "24";
        }
    },
    beforeUnmount() {
        if (this.chartDataFetchInterval) {
            clearInterval(this.chartDataFetchInterval);
        }
    },
    methods: {
        // Get color of bar chart for this datapoint
        getBarColorForDatapoint(datapoint) {
            if (datapoint.maintenance != null) {
                // Target is in maintenance
                return "rgba(23,71,245,0.41)";
            } else if (datapoint.down === 0) {
                // Target is up, no need to display a bar
                return "#000";
            } else if (datapoint.up === 0) {
                // Target is down
                return "rgba(220, 53, 69, 0.41)";
            } else {
                // Show yellow for mixed status
                return "rgba(245, 182, 23, 0.41)";
            }
        },
        // push datapoint to chartData
        pushDatapoint(datapoint, avgPingData, minPingData, maxPingData, downData, colorData) {
            const x = this.$root.unixToDateTime(datapoint.timestamp);

            // Show ping values if it was up in this period
            avgPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing > 0 ? datapoint.avgPing : null,
            });
            minPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing > 0 ? datapoint.minPing : null,
            });
            maxPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing > 0 ? datapoint.maxPing : null,
            });
            downData.push({
                x,
                y: datapoint.down + (datapoint.maintenance || 0),
            });

            colorData.push(this.getBarColorForDatapoint(datapoint));
        },
        // get the average of a set of datapoints
        getAverage(datapoints) {
            const totalUp = datapoints.reduce((total, current) => total + current.up, 0);
            const totalDown = datapoints.reduce((total, current) => total + current.down, 0);
            const totalMaintenance = datapoints.reduce((total, current) => total + (current.maintenance || 0), 0);
            const totalPing = datapoints.reduce((total, current) => total + current.avgPing * current.up, 0);
            const minPing = datapoints.reduce((min, current) => Math.min(min, current.minPing), Infinity);
            const maxPing = datapoints.reduce((max, current) => Math.max(max, current.maxPing), 0);

            // Find the middle timestamp to use
            let midpoint = Math.floor(datapoints.length / 2);

            return {
                timestamp: datapoints[midpoint].timestamp,
                up: totalUp,
                down: totalDown,
                maintenance: totalMaintenance > 0 ? totalMaintenance : undefined,
                avgPing: totalUp > 0 ? totalPing / totalUp : 0,
                minPing,
                maxPing,
            };
        },
        getChartDatapointsFromHeartbeatList() {
            // Render chart using heartbeatList
            let lastHeartbeatTime;
            const monitorInterval = this.$root.monitorList[this.monitorId]?.interval;
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
        if (!this.customStartDate || !this.customEndDate) {
            toast.error("Veuillez sélectionner une date de début et une date de fin.");
            return;
        }

        this.loading = true;

        try {
            const res = await fetch(`http://localhost:3001/api/status-by-period?start=${this.customStartDate}&end=${this.customEndDate}&statuses=1&monitorId=${this.monitorId}`);

            // Vérification du type de réponse
            const contentType = res.headers.get("content-type");
            if (!res.ok || !contentType || !contentType.includes("application/json")) {
                const errorText = await res.text();
                console.error("⚠️ Réponse non-JSON :", errorText);
                toast.error("Le serveur a renvoyé une réponse invalide.");
                this.loading = false;
                return;
            }

            const json = await res.json();

            if (!json.success) {
                toast.error(json.message || "Erreur de récupération des données");
            } else {
                this.heartbeatList = json.data;
                this.chartPeriodHrs = 0; 
            }
        } catch (e) {
            toast.error("Erreur réseau");
            console.error("Erreur dans filtergraphByPeriod:", e);
        }

        this.loading = false;
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
            }

            return {
                datasets: [
                    {
                        // average ping chart
                        data: avgPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderColor: "#5CDD8B",
                        backgroundColor: "#5CDD8B06",
                        yAxisID: "y",
                        label: "avg-ping",
                    },
                    {
                        // minimum ping chart
                        data: minPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderColor: "#3CBD6B38",
                        backgroundColor: "#5CDD8B06",
                        yAxisID: "y",
                        label: "min-ping",
                    },
                    {
                        // maximum ping chart
                        data: maxPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderColor: "#7CBD6B38",
                        backgroundColor: "#5CDD8B06",
                        yAxisID: "y",
                        label: "max-ping",
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
                color: $dark-font-color;
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
