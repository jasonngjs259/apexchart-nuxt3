<template>
  <div class="stock-chart-wrapper">
    <div class="controls">
      <div id="time-period-btns">
        <button
          v-for="p in periods"
          :key="p"
          class="time-btn"
          :class="{ active: currentPeriod === p }"
          @click="handlePeriodChange(p)"
        >
          {{ p }}
        </button>
      </div>

      <div class="interval-row">
        <label>Interval</label>
        <select
          id="interval-select"
          v-model="currentInterval"
          @change="handleIntervalChange($event)"
        >
          <option value="1min">1min</option>
          <option value="5min">5min</option>
          <option value="15min">15min</option>
          <option value="30min">30min</option>
          <option value="1hour">1hour</option>
          <option value="4hour">4hour</option>
          <option value="1day">1day</option>
          <option value="1week">1week</option>
          <option value="1month">1month</option>
        </select>

        <button id="theme-toggle" @click="toggleTheme">Toggle Theme</button>
      </div>

      <div
        id="date-range-container"
        v-if="currentPeriod === 'custom'"
        class="date-range"
      >
        <input id="start-date" type="date" v-model="customDateRange.start" />
        <input id="end-date" type="date" v-model="customDateRange.end" />
        <button id="apply-range" @click="applyCustomDateRange">Apply</button>
      </div>
    </div>

    <div id="loading-indicator" v-show="isLoading" class="loading">
      Loading...
    </div>
    <div id="chart"></div>
  </div>
</template>

<script>
import ApexStock from "apexstock";

export default {
  data() {
    return {
      periods: ["1D", "5D", "1M", "3M", "6M", "1Y", "5Y", "ALL", "custom"],
      currentPeriod: "1Y",
      currentInterval: "1day",
      isLoading: false,
      historicalData: {},
      masterData: null,
      allData: [],
      customDateRange: { start: null, end: null },
      stockChart: null,
    };
  },
  methods: {
    formatDateForInput(date) {
      return date.toISOString().split("T")[0];
    },
    toggleLoadingIndicator(val) {
      this.isLoading = val;
    },
    generateMasterDataset() {
      if (this.masterData) return;

      const tenYearsAgo = new Date();
      tenYearsAgo.setFullYear(tenYearsAgo.getFullYear() - 10);
      const now = new Date();
      let baseDate = tenYearsAgo.getTime();
      const endDate = now.getTime();
      const timeRange = endDate - baseDate;

      const count = 2500;
      const data = [];
      let price = 100;
      const volMin = 6000;
      const volMax = 900000;

      for (let i = 0; i < count; i++) {
        const position = i / (count - 1);
        const timestamp = baseDate + position * timeRange;
        const currentDate = new Date(timestamp);
        const dayOfWeek = currentDate.getDay();
        if (dayOfWeek === 0 || dayOfWeek === 6) continue;

        const volatilityFactor = 2.0;
        const change = (Math.random() - 0.5) * volatilityFactor;
        const open = price;
        const close = price + change;
        const high =
          Math.max(open, close) + Math.random() * (volatilityFactor / 2);
        const low =
          Math.min(open, close) - Math.random() * (volatilityFactor / 2);
        const volumeFactor = (Math.abs(change) / volatilityFactor) * 3;
        const volumeVariance = Math.random() * 0.5 + 0.5;
        const volume = Math.floor(
          volMin + (volMax - volMin) * volumeFactor * volumeVariance
        );

        data.push({
          x: new Date(currentDate).toString(),
          y: [
            Number(open.toFixed(2)),
            Number(high.toFixed(2)),
            Number(low.toFixed(2)),
            Number(close.toFixed(2)),
          ],
          v: volume,
        });
        price = close;
      }

      this.masterData = data;
    },
    getBaseDateForPeriod(period) {
      const now = new Date();
      switch (period) {
        case "1D":
          return new Date(
            now.getFullYear(),
            now.getMonth(),
            now.getDate(),
            9,
            30,
            0
          ).getTime();
        case "5D":
          const fiveDaysAgo = new Date(now);
          fiveDaysAgo.setDate(fiveDaysAgo.getDate() - 7);
          return new Date(
            fiveDaysAgo.getFullYear(),
            fiveDaysAgo.getMonth(),
            fiveDaysAgo.getDate(),
            9,
            30,
            0
          ).getTime();
        case "1M":
          const oneMonthAgo = new Date(now);
          oneMonthAgo.setMonth(oneMonthAgo.getMonth() - 1);
          return oneMonthAgo.getTime();
        case "3M":
          const threeMonthsAgo = new Date(now);
          threeMonthsAgo.setMonth(threeMonthsAgo.getMonth() - 3);
          return threeMonthsAgo.getTime();
        case "6M":
          const sixMonthsAgo = new Date(now);
          sixMonthsAgo.setMonth(sixMonthsAgo.getMonth() - 6);
          return sixMonthsAgo.getTime();
        case "1Y":
          const oneYearAgo = new Date(now);
          oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);
          return oneYearAgo.getTime();
        case "5Y":
          const fiveYearsAgo = new Date(now);
          fiveYearsAgo.setFullYear(fiveYearsAgo.getFullYear() - 5);
          return fiveYearsAgo.getTime();
        case "ALL":
          const tenYearsAgo = new Date(now);
          tenYearsAgo.setFullYear(tenYearsAgo.getFullYear() - 10);
          return tenYearsAgo.getTime();
        default:
          const defaultDate = new Date(now);
          defaultDate.setFullYear(defaultDate.getFullYear() - 1);
          return defaultDate.getTime();
      }
    },
    aggregateDataForInterval(data, interval) {
      const result = [];
      if (interval === "1week") {
        let currentWeek = null;
        let weekData = null;
        for (const point of data) {
          const date = new Date(point.x);
          const weekStart = new Date(date);
          weekStart.setDate(date.getDate() - date.getDay() + 1);
          if (!currentWeek || weekStart > currentWeek) {
            if (weekData) result.push(weekData);
            currentWeek = weekStart;
            weekData = {
              x: weekStart.toString(),
              y: [point.y[0], point.y[1], point.y[2], point.y[3]],
              v: point.v,
            };
          } else {
            weekData.y[1] = Math.max(weekData.y[1], point.y[1]);
            weekData.y[2] = Math.min(weekData.y[2], point.y[2]);
            weekData.y[3] = point.y[3];
            weekData.v += point.v;
          }
        }
        if (weekData) result.push(weekData);
      } else if (interval === "1month") {
        let currentMonth = null;
        let monthData = null;
        for (const point of data) {
          const date = new Date(point.x);
          const monthStart = new Date(date.getFullYear(), date.getMonth(), 1);
          if (!currentMonth || monthStart > currentMonth) {
            if (monthData) result.push(monthData);
            currentMonth = monthStart;
            monthData = {
              x: monthStart.toString(),
              y: [point.y[0], point.y[1], point.y[2], point.y[3]],
              v: point.v,
            };
          } else {
            monthData.y[1] = Math.max(monthData.y[1], point.y[1]);
            monthData.y[2] = Math.min(monthData.y[2], point.y[2]);
            monthData.y[3] = point.y[3];
            monthData.v += point.v;
          }
        }
        if (monthData) result.push(monthData);
      }
      return result;
    },
    expandDataForIntraday(data, interval) {
      const result = [];
      const candlesPerDay = {
        "1min": 390,
        "5min": 78,
        "15min": 26,
        "30min": 13,
        "1hour": 7,
        "4hour": 2,
      }[interval];
      for (const dayCandle of data) {
        const dayDate = new Date(dayCandle.x);
        const dayOpen = dayCandle.y[0];
        const dayHigh = dayCandle.y[1];
        const dayLow = dayCandle.y[2];
        const dayClose = dayCandle.y[3];
        const dayVolume = dayCandle.v;
        const marketOpen = new Date(dayDate);
        marketOpen.setHours(9, 30, 0, 0);
        let currentPrice = dayOpen;
        let prevPrice = dayOpen;
        for (let i = 0; i < candlesPerDay; i++) {
          const candleTime = new Date(marketOpen);
          switch (interval) {
            case "1min":
              candleTime.setMinutes(marketOpen.getMinutes() + i);
              break;
            case "5min":
              candleTime.setMinutes(marketOpen.getMinutes() + i * 5);
              break;
            case "15min":
              candleTime.setMinutes(marketOpen.getMinutes() + i * 15);
              break;
            case "30min":
              candleTime.setMinutes(marketOpen.getMinutes() + i * 30);
              break;
            case "1hour":
              candleTime.setHours(marketOpen.getHours() + i);
              break;
            case "4hour":
              candleTime.setHours(marketOpen.getHours() + i * 4);
              break;
          }
          const hours = candleTime.getHours();
          const minutes = candleTime.getMinutes();
          const timeInMinutes = hours * 60 + minutes;
          if (timeInMinutes < 570 || timeInMinutes > 960) continue;
          const isLastCandle = i === candlesPerDay - 1;
          if (isLastCandle) currentPrice = dayClose;
          else {
            const volatilityFactor = {
              "1min": 0.2,
              "5min": 0.3,
              "15min": 0.5,
              "30min": 0.7,
              "1hour": 1.0,
              "4hour": 1.5,
            }[interval];
            const progress = i / (candlesPerDay - 1);
            const drift = (dayClose - dayOpen) * 0.1 * progress;
            const randomComponent = (Math.random() - 0.5) * volatilityFactor;
            currentPrice = prevPrice + drift + randomComponent;
            currentPrice = Math.min(Math.max(currentPrice, dayLow), dayHigh);
          }
          const open = prevPrice;
          const close = currentPrice;
          const high = Math.max(open, close) + Math.random() * 0.1;
          const low = Math.min(open, close) - Math.random() * 0.1;
          const canHigh = Math.min(high, dayHigh);
          const canLow = Math.max(low, dayLow);
          const volumePortion = dayVolume / candlesPerDay;
          const volumeVariance = 0.5 + Math.random();
          const volume = Math.floor(volumePortion * volumeVariance);
          result.push({
            x: candleTime.toString(),
            y: [
              Number(open.toFixed(2)),
              Number(canHigh.toFixed(2)),
              Number(canLow.toFixed(2)),
              Number(close.toFixed(2)),
            ],
            v: volume,
          });
          prevPrice = currentPrice;
        }
      }
      return result;
    },
    extractDataForPeriod(period, interval) {
      if (!this.masterData) this.generateMasterDataset();
      const baseDate = this.getBaseDateForPeriod(period);
      const now = new Date().getTime();
      let filteredData = this.masterData.filter((point) => {
        const pointDate = new Date(point.x).getTime();
        return pointDate >= baseDate && pointDate <= now;
      });
      if (
        period === "custom" &&
        this.customDateRange.start &&
        this.customDateRange.end
      ) {
        const startDate = new Date(this.customDateRange.start).getTime();
        const endDate = new Date(this.customDateRange.end).getTime();
        filteredData = this.masterData.filter((point) => {
          const pointDate = new Date(point.x).getTime();
          return pointDate >= startDate && pointDate <= endDate;
        });
      }
      if (interval !== "1day") {
        if (interval === "1week" || interval === "1month")
          filteredData = this.aggregateDataForInterval(filteredData, interval);
        else if (
          ["1min", "5min", "15min", "30min", "1hour", "4hour"].includes(
            interval
          )
        )
          filteredData = this.expandDataForIntraday(filteredData, interval);
      }
      return filteredData;
    },
    async fetchData(period, interval) {
      this.toggleLoadingIndicator(true);
      await new Promise((r) => setTimeout(r, 600));
      try {
        const newData = this.extractDataForPeriod(period, interval);
        const cacheKey = `${period}_${interval}`;
        this.historicalData[cacheKey] = newData;
        this.updateChart(newData);
      } catch (err) {
        console.error("fetchData error", err);
      } finally {
        this.toggleLoadingIndicator(false);
      }
    },
    updateChart(newData) {
      this.allData = newData;
      if (!this.stockChart) return;
      this.stockChart.update({ series: [{ data: newData }] });
    },
    handlePeriodChange(period) {
      this.currentPeriod = period;
      if (period !== "custom") {
        this.autoSelectInterval(period);
        this.fetchData(period, this.currentInterval);
      }
    },
    autoSelectInterval(period) {
      switch (period) {
        case "1D":
          this.currentInterval = "15min";
          break;
        case "5D":
          this.currentInterval = "1hour";
          break;
        case "1M":
          this.currentInterval = "4hour";
          break;
        case "3M":
        case "6M":
        case "1Y":
          this.currentInterval = "1day";
          break;
        case "5Y":
        case "ALL":
          this.currentInterval = "1week";
          break;
        default:
          this.currentInterval = "1day";
      }
    },
    handleIntervalChange(e) {
      const val = e && e.target ? e.target.value : this.currentInterval;
      this.currentInterval = val;
      if (this.currentPeriod !== "custom")
        this.fetchData(this.currentPeriod, val);
    },
    applyCustomDateRange() {
      if (!this.customDateRange.start || !this.customDateRange.end) {
        alert("Please select both start and end dates");
        return;
      }
      this.fetchData("custom", this.currentInterval);
    },
    toggleTheme() {
      document.body.classList.toggle("dark-mode");
      const isDark = document.body.classList.contains("dark-mode");
      if (this.stockChart)
        this.stockChart.update({ theme: { mode: isDark ? "dark" : "light" } });
    },
  },
  mounted() {
    this.generateMasterDataset();
    this.allData = this.extractDataForPeriod(
      this.currentPeriod,
      this.currentInterval
    );

    const chartOptions = {
      chart: { height: 400 },
      series: [{ data: this.allData }],
      xaxis: { type: "category" },
      yaxis: { tooltip: { enabled: true } },
      theme: {
        mode: document.body.classList.contains("dark-mode") ? "dark" : "light",
      },
    };

    this.stockChart = new ApexStock(
      document.querySelector("#chart"),
      chartOptions
    );
    this.stockChart.render();

    const endDate = new Date();
    const startDate = new Date();
    startDate.setDate(startDate.getDate() - 30);
    this.customDateRange.start = this.formatDateForInput(startDate);
    this.customDateRange.end = this.formatDateForInput(endDate);
  },
};
</script>

<style scoped>
.stock-chart-wrapper {
  max-width: 1000px;
  margin: 0 auto;
}
.controls {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 8px;
}
#time-period-btns .time-btn {
  margin-right: 6px;
}
.interval-row {
  display: flex;
  gap: 8px;
  align-items: center;
}
.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 40px;
}
#chart {
  min-height: 380px;
}
</style>
