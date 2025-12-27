<script>
import ApexStock from "apexstock";

export default {
  data() {
    return {
      chartOptions: {
        chart: {
          height: 400,
        },
        series: [
          {
            data: [
              {
                x: "2024-01-01",
                y: [100, 101.05, 96.53, 98],
                v: 363305,
              },
              {
                x: "2024-01-02",
                y: [98, 98.39, 95.55, 97.51],
                v: 612318,
              },
              {
                x: "2024-01-03",
                y: [97.51, 98.14, 97.29, 97.63],
                v: 915487,
              },
            ],
          },
        ],
        theme: {
          mode: "dark", // 'light' or 'dark'
        },

        // plotOptions: {
        //   stockChart: {
        //     indicators: {
        //       // Configure available indicators (Overlays - drawn on main chart)
        //       "moving average": { enabled: true },
        //       "bollinger bands": { enabled: true },
        //       "exponential moving average": { enabled: true },
        //       "fibonacci retracements": { enabled: true },
        //       "linear regression": { enabled: true },
        //       "ichimoku cloud indicator": { enabled: true },

        //       // Oscillators (displayed in separate panels)
        //       rsi: { enabled: true },
        //       macd: { enabled: true },
        //       volumes: { enabled: true },
        //       "price volume trend": { enabled: true },
        //       "stochastic oscillator": { enabled: true },
        //       "standard deviation indicator": { enabled: true },
        //       "average directional index": { enabled: true },
        //       "chaikin oscillator": { enabled: true },
        //       "commodity channel index": { enabled: true },
        //       "trend strength index": { enabled: true },
        //       "accelerator oscillator": { enabled: true },
        //       "bollinger bands %b": { enabled: true },
        //       "bollinger bands width": { enabled: true },
        //     },
        //   },
        // },
      },
    };
  },
  mounted() {
    // Create and render the ApexStock chart
    const stockChart = new ApexStock(
      document.querySelector("#chart"),
      this.chartOptions
    );

    stockChart.render();
  },
};
</script>

<template>
  <div id="chart"></div>
</template>
