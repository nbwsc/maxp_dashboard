<template>
  <div class="hello">
    <div>
      <el-date-picker
        v-model="date"
        type="date"
        placeholder="Pick a day"
        @change="refresh"
      >
      </el-date-picker>
      <el-time-picker
        v-model="timeStart"
        placeholder="开始时间"
        @change="refresh"
      >
      </el-time-picker>
      <el-time-picker
        v-model="timeEnd"
        placeholder="结束时间"
        @change="refresh"
      >
      </el-time-picker>
      <el-select v-model="gridType" :disabled="true">
        <el-option
          v-for="item in gridOptions"
          :key="item"
          :label="item"
          :value="item"
        >
        </el-option>
      </el-select>

      <el-button type="success" size="default" @click="refresh">刷新</el-button>
    </div>
    <div class="chartpanel">
      <div class="chart">
        <h3>缓存命中率</h3>
        <v-chart class="chart1" :option="option1" />
      </div>
      <div class="chart">
        <h3>HTTP请求状态</h3>
        <v-chart class="chart1" :option="option2" />
      </div>
      <div class="chart">
        <h3>请求响应时长</h3>
        <v-chart class="chart1" :option="option3" />
      </div>
      <div class="chart">
        <h3>流量带宽</h3>
        <v-chart class="chart1" :option="option4" />
      </div>
      <div class="chart">
        <h3>访问次数</h3>
        <v-chart class="chart1" :option="option5" />
      </div>
    </div>
  </div>
</template>

<script>
import request from "axios";
import moment from "moment";

import { use } from "echarts/core";
import { CanvasRenderer } from "echarts/renderers";
import { PieChart, BarChart } from "echarts/charts";
import {
  TitleComponent,
  TooltipComponent,
  LegendComponent,
} from "echarts/components";
import VChart, { THEME_KEY } from "vue-echarts";
import _ from "lodash";

use([
  CanvasRenderer,
  PieChart,
  BarChart,
  TitleComponent,
  TooltipComponent,
  LegendComponent,
]);

export default {
  name: "Dashboard",
  data() {
    return {
      date: new Date("2021-12-23"),
      timeStart: moment("2021-12-23 07:00:00").toDate(),
      timeEnd: moment("2021-12-23 08:00:00").toDate(),
      provide: {
        [THEME_KEY]: "dark",
      },
      gridType: "by 秒",
      gridOptions: ["by 秒", "by 分钟", "by 小时"],
      option1: {},
      option2: {},
    };
  },
  components: {
    VChart,
  },
  created() {
    this.refresh();
  },
  methods: {
    async refresh() {
      const d = moment(this.timeEnd).diff(moment(this.timeStart), "second");
      if (d > 3600) {
        this.gridType = "by 小时";
      } else if (d > 60) {
        this.gridType = "by 分钟";
      } else {
        this.gridType = "by 秒";
      }
      this.loading = this.$loading();
      const timeStr = moment(this.date).format("YYYY-MM-DD");
      const startStr = moment(this.timeStart).format("hh:mm:ss");
      const endStr = moment(this.timeEnd).format("hh:mm:ss");
      const dIdRes = await request.get(
        `https://36hudls5ed.execute-api.cn-northwest-1.amazonaws.com.cn/getDashboardData\
?date=${timeStr}&timeStart=${startStr}&timeEnd=${endStr}`
      );
      // https://simplifyresults.s3.cn-northwest-1.amazonaws.com.cn/c504ca86-efee-4db5-a041-bdfb6079ee23.csv
      // const test = "55bd2a2a-57f6-4ad1-84dd-9b40ecb6516f";
      // await this.getDashboardData(test, 0);
      await this.getDashboardData(dIdRes.data.data, 0);
    },
    async getDashboardData(id, retry) {
      if (retry > 30) {
        return this.$message({
          message: "请求超时",
          type: "info",
        });
      }
      const dataUrl = `https://simplifyresults.s3.cn-northwest-1.amazonaws.com.cn/${id}.csv`;
      try {
        const dashboardRow = await request.get(dataUrl);
        const txt = dashboardRow.data;
        this.parseData(txt);
      } catch (error) {
        if (error.message.includes("404")) {
          retry += 1;
          setTimeout(() => {
            this.getDashboardData(id, retry);
          }, 1000);
        } else if (error.message.includes("request")) {
          this.$message({
            message: "请求失败:" + error.message,
            type: "info",
          });
        } else {
          this.$message({
            message: error.message,
            type: "info",
          });
        }
      }
    },

    parseData(txt) {
      const rows = txt.split("\n");
      // const keys = JSON.parse(`[${rows[0]}]`);
      this.logData = [];
      for (const row of rows.slice(1)) {
        try {
          const dArr = JSON.parse(`[${row.slice(0, -1)}]`);
          if (dArr.length) {
            this.logData.push(dArr);
          }
        } catch (error) {
          console.error(error);
        }
      }
      this.groupedData = _.groupBy(this.logData, (e) => {
        switch (this.gridType) {
          case "by 小时":
            return e[1].slice(0, 2);
          case "by 分钟":
            return e[1].slice(3, 5);
          case "by 秒":
            return e[1].slice(6, 8);
        }
      });
      this.xData = Object.keys(this.groupedData).sort((a, b) => {
        return +a - b;
      });
      this.updateChart1();
      this.updateChart2();
      this.updateChart3();
      this.updateChart4();
      this.updateChart5();
      this.loading.close();
    },
    _fixXAxis(d) {
      switch (this.gridType) {
        case "by 小时":
          return `${d}:00`;
        case "by 分钟":
          return `hh:${d}`;
        case "by 秒":
          return d;
      }
    },
    updateChart1() {
      const data = [];
      for (const x of this.xData) {
        const rows = this.groupedData[x];
        const res =
          (rows.filter((row) => row[22] === "Hit").length / rows.length) * 100;
        data.push(res);
      }
      this.option1 = {
        xAxis: {
          type: "category",
          data: this.xData.map(this._fixXAxis),
        },
        tooltip: {
          trigger: "axis",
          axisPointer: {
            type: "shadow",
          },
        },
        yAxis: {
          type: "value",
        },
        series: [
          {
            data,
            type: "bar",
          },
        ],
      };
    },
    updateChart2() {
      // httpstatus
      const statusMap = {};
      for (const row of this.logData) {
        const status = row[8];
        if (!status) {
          console.log(row);
        }
        if (!statusMap[status]) {
          statusMap[status] = 1;
        } else {
          statusMap[status] += 1;
        }
      }
      const data = [];
      for (const k in statusMap) {
        data.push({ value: statusMap[k], name: k });
      }
      this.option2 = {
        tooltip: {
          trigger: "item",
        },
        legend: {
          orient: "vertical",
          left: "left",
          data: Object.keys(statusMap),
        },
        series: [
          {
            name: "Traffic Sources",
            type: "pie",
            radius: "55%",
            center: ["50%", "60%"],
            data,
            emphasis: {
              itemStyle: {
                shadowBlur: 10,
                shadowOffsetX: 0,
                shadowColor: "rgba(0, 0, 0, 0.5)",
              },
            },
          },
        ],
      };
      this.$forceUpdate();
    },
    _getIndex(arr, n) {
      return Math.floor(n * arr.length);
    },
    updateChart3() {
      // 18
      const data = [];
      const l1 = [];
      const l2 = [];
      const l3 = [];
      for (const x of this.xData) {
        const rows = this.groupedData[x];
        const sorted = rows
          .map((e) => e[18])
          .sort((a, b) => {
            return a - b;
          });

        const n90 = sorted[this._getIndex(sorted, 0.9)];
        const n95 = sorted[this._getIndex(sorted, 0.95)];
        const nmax = sorted[sorted.length - 1];
        l1.push(n90);
        l2.push(n95);
        l3.push(nmax);
        data.push(rows.length);
      }
      this.option3 = {
        xAxis: {
          type: "category",
          data: this.xData.map(this._fixXAxis),
        },
        yAxis: {
          type: "value",
        },
        legend: {
          data: ["P90", "P95", "MAX"],
        },
        series: [
          {
            name: "P90",
            data: l1,
            type: "line",
          },
          {
            name: "P95",
            data: l2,
            type: "line",
          },
          {
            name: "MAX",
            data: l3,
            type: "line",
          },
        ],
      };
    },

    updateChart4() {
      const data = [];
      for (const x of this.xData) {
        let sum = 0;
        const rows = this.groupedData[x];
        for (const row of rows) {
          sum += +row[3];
        }
        data.push(sum / 1000);
      }
      this.option4 = {
        xAxis: {
          type: "category",
          data: this.xData.map(this._fixXAxis),
        },
        tooltip: {
          trigger: "axis",
          axisPointer: {
            type: "shadow",
          },
        },
        yAxis: {
          type: "value",
        },
        series: [
          {
            data,
            type: "bar",
          },
        ],
      };
    },
    updateChart5() {
      const data = [];
      for (const x of this.xData) {
        const rows = this.groupedData[x];
        data.push(rows.length);
      }
      this.option5 = {
        xAxis: {
          type: "category",
          data: this.xData.map(this._fixXAxis),
        },
        tooltip: {
          trigger: "axis",
          axisPointer: {
            type: "shadow",
          },
        },
        yAxis: {
          type: "value",
        },
        series: [
          {
            data,
            type: "bar",
          },
        ],
      };
    },
  },
};
</script>

<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.chartpanel {
  display: flex;
  flex-wrap: wrap;
}
.chart {
  width: 45%;
  padding: 16px;
}
.chart1 {
  height: 300px;
}
</style>
