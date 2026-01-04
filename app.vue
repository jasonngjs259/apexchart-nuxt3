<template>
  <div>
    <ClientOnly>
      <StockChart />
    </ClientOnly>
    <ClientOnly>
      <BarChart />
    </ClientOnly>
  </div>
</template>

<!-- <script>
import ApexCharts from "apexcharts";

export default {
  setup() {
    window.ApexCharts = ApexCharts;
  },
  mounted() {},
};
</script> -->
