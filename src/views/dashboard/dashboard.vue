<template>
  <div id="tet">
    <Vtitle head="主机监视" />
    <div class="mx-3 shadow-lg p-3 mb-5 bg-body rounded" id="dash">
      <div class="row mt-3">
        <div class="col-9">
          <div class="card">
            <div class="card-header">
              <span>
                <svg class="bi me-2" width="24" height="24">
                  <use xlink:href="#cpu" />
                </svg>
                硬件监视</span
              >
            </div>
            <div class="card-body p-4" style="height: 320px">
              <div class="row">
                <div class="col-6">
                  <div class="text-center">操作系统：{{ hardware.system }}</div>
                </div>
                <div class="col-6">
                  <div class="text-center">
                    开机时间：{{ hardware.setuptime }}
                  </div>
                </div>

                <div class="col-3">
                  <div
                    id="cpuTemp"
                    ref="cpuTemp"
                    style="width: 260px; height: 170px"
                  ></div>
                </div>
                <div class="col-3">
                  <div
                    id="cpuUsage"
                    ref="cpuUsage"
                    style="width: 260px; height: 170px"
                  ></div>
                </div>
                <div class="col-3">
                  <div
                    id="memUsage"
                    ref="memUsage"
                    style="width: 260px; height: 170px"
                  ></div>
                </div>
                <div class="col-3">
                  <div
                    id="diskUsage"
                    ref="diskUsage"
                    style="width: 260px; height: 170px"
                  ></div>
                </div>

                <div
                  class="col-3 mt-3"
                  v-for="(item, index) in stat"
                  :key="index"
                >
                  <div class="stat-info__item">
                    <div
                      class="stat-info__icon"
                      :style="{ 'background-color': item.bgColor }"
                    >
                      <i :class="'fas ' + item.icon"></i>
                    </div>
                    <div class="stat-info__detail">
                      <span class="stat-info__total">{{ item.total }}</span>
                      <span class="stat-info__title">{{ item.title }}</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="col-3">
          <div class="card">
            <div class="card-header">
              <span>
                <svg class="bi me-2" width="24" height="24">
                  <use xlink:href="#bell" /></svg
                >报警事件（Top5）</span
              >
            </div>
            <div class="card-body" style="height: 320px">
              <div class="row">
                <div class="col-12">
                  <!-- <div class="btn-group mb-3" role="group">
                <button
                  type="button"
                  class="btn btn-info"
                  @click="initWebSocket"
                >
                  打开
                </button>
                <button type="button" class="btn btn-danger" @click="close">
                  关闭
                </button>
                <button type="button" class="btn btn-warning" @click="clear">
                  清空
                </button>
              </div> -->
                  <!-- 
              <div class="input-group mb-3">
                <input
                  id="wssend"
                  title="wssend"
                  type="text"
                  class="form-control"
                  aria-describedby="button-addon2"
                  v-model="input"
                />
                <button
                  class="btn btn-outline-info"
                  type="button"
                  id="button-addon2"
                  @click="send"
                >
                  <svg class="bi" width="1em" height="1em">
                    <use xlink:href="#send" /></svg
                  >send
                </button>
              </div> -->

                  <textarea
                    title="wsmsg"
                    class="form-control"
                    rows="11"
                    v-model="msg"
                  >
                  </textarea>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from "vue";
import { useStore } from "vuex";
import axios from "axios";
import * as echarts from "echarts";
import Vtitle from "../../components/Vtitle.vue";

const store = useStore();
const th = ["ID", "名称", "PID", "CPU(%)", "内存(%)", "启动时间", "操作"];
const path = "ws://127.0.0.1:8002/echo";
const hardware = reactive({
  system: "",
  setuptime: "",
});
const timer = ref();
const msg = ref("");
const input = ref("");
const cpuTemp = ref(null);
const cpuUsage = ref(null);
const memUsage = ref(null);
const diskUsage = ref(null);
let TempChart = reactive({});
let CpuChart = reactive({});
let MemChart = reactive({});
let DiskChart = reactive({});
let activities = reactive({});
const stat = reactive([]);
let ws = {};
onMounted(() => {
  initCharts();
  loadData();
  // loadMoniP();
  initWebSocket(); // 初始化weosocket，发起连接
  // 这里使用定时器是为了等待连接后才发送数据，避免错误
  setTimeout(() => {
    // 添加状态判断，当为OPEN时，前端发送消息给后端
    if (ws.readyState === 1) {
      // 把后台需要的参数数据传过去
      const obj = {
        topic: "alarm",
        cycle: 5,
      };
      // 发给后端的数据需要字符串化
      wssend(obj);
    }
  }, 500);
  timer.value = setInterval(() => {
    loadData();
    // loadMoniP();
  }, 5000);
});
function initWebSocket() {
  ws = new WebSocket(path);
  ws.onopen = () => {
    console.log("ws连接状态：" + ws.readyState);
  };
  ws.onclose = () => {
    ws = null;
  };
  ws.onmessage = (evt) => {
    // const dd = eval("(" + evt.data + ")");
    // // const dataJson = JSON.parse(evt.data);
    // if (dd.length > 0) {
    //   const dataArray = dataJson.map((item) => {
    //     return {
    //       content: item.content,
    //       timestamp: parseTime(item.timestamp),
    //     };
    //   });
    //   activities = dataArray.reverse().slice(0, 5);
    // } else {
    msg.value = evt.data + "\r\n";
    // }
  };
  ws.onerror = (evt) => {
    msg.value = "ERROR: " + evt.data;
  };
}

function send() {
  if (!ws) {
    return false;
  }
  ws.send(input.value);
}

function close() {
  if (!ws) {
    return false;
  }
  ws.close();
}
function clear() {
  msg.value = "";
}

function wssend(data) {
  // 数据发送
  ws.send(JSON.stringify(data));
}

function gaugeimg(id, title, min, max, val, unit) {
  const option = {
    title: {
      show: false,
      text: title,
      x: "center",
      y: "90%",
    },
    tooltip: {
      formatter: "{b} : {c}" + unit,
      confine: true,
    },
    toolbox: {
      show: false,
      feature: {
        mark: {
          show: true,
        },
        restore: {
          show: true,
        },
        saveAsImage: {
          show: true,
        },
      },
    },
    series: [
      {
        center: ["50%", "50%"],
        number: [0, "50%"],
        // 仪表盘起始角度
        startAngle: 240,
        // 仪表盘结束角度
        endAngle: -60,
        name: title,
        type: "gauge",
        radius: "85%",
        // 坐标轴线
        axisLine: {
          // 属性lineStyle控制线条样式
          lineStyle: {
            color: [
              [0.25, "#ddd"],
              [1, "#ddd"],
            ],
            width: 10,
          },
        },
        // 坐标轴小标记
        axisTick: {
          show: false,
        },
        // 坐标轴文本标签，详见axis.axisLabel
        axisLabel: {
          show: true,
          distance: 1,
        },
        // 分隔线
        splitLine: {
          // 默认显示，属性show控制显示与否
          show: false,
        },
        // 指针粗细
        pointer: {
          width: 5,
        },
        title: {
          // 其余属性默认使用全局文本样式，详见TEXTSTYLE
          fontWeight: "bolder",
          offsetCenter: [0, "-20%"],
          padding: [5, 5],
          fontSize: 12,
        },
        detail: {
          formatter: "{value}" + unit,
          // 其余属性默认使用全局文本样式，详见TEXTSTYLE
          color: "inherit",
          fontWeight: "bolder",
          fontSize: 15,
          offsetCenter: [0, "20%"],
        },
        data: [{}],
      },
    ],
  };
  option.series[0].data[0].value = val;
  option.series[0].data[0].name = title;
  option.series[0].axisLine.lineStyle.color[0][0] = (val - min) / (max - min);
  option.series[0].axisLine.lineStyle.color[0][1] = detectionData(val, id);
  switch (id) {
    case "cpuTemp":
      TempChart.setOption(option);
      break;
    case "cpuUsage":
      CpuChart.setOption(option);
      break;
    case "memUsage":
      MemChart.setOption(option);
      break;
    case "diskUsage":
      DiskChart.setOption(option);
      break;
  }
}

function detectionData(str, id) {
  if (id === "cpuTemp") {
    var color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
      {
        offset: 0,
        color: "#32cd32",
      },
      {
        offset: 1,
        color: "#32cd32",
      },
    ]);
    if (str >= 60 && str <= 80) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff8c00",
        },
        {
          offset: 1,
          color: "#ff8c00",
        },
      ]);
    }
    if (str > 80) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff0000",
        },
        {
          offset: 1,
          color: "#ff0000",
        },
      ]);
    }
    return color;
  }
  if (id === "cpuUsage") {
    color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
      {
        offset: 0,
        color: "#32cd32",
      },
      {
        offset: 1,
        color: "#32cd32",
      },
    ]);
    if (str >= 70 && str <= 90) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff8c00",
        },
        {
          offset: 1,
          color: "#ff8c00",
        },
      ]);
    }
    if (str > 90) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff0000",
        },
        {
          offset: 1,
          color: "#ff0000",
        },
      ]);
    }
    return color;
  }
  if (id === "memUsage") {
    color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
      {
        offset: 0,
        color: "#32cd32",
      },
      {
        offset: 1,
        color: "#32cd32",
      },
    ]);
    if (str >= 70 && str <= 90) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff8c00",
        },
        {
          offset: 1,
          color: "#ff8c00",
        },
      ]);
    }
    if (str > 90) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff0000",
        },
        {
          offset: 1,
          color: "#ff0000",
        },
      ]);
    }
    return color;
  }
  if (id === "diskUsage") {
    color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
      {
        offset: 0,
        color: "#32cd32",
      },
      {
        offset: 1,
        color: "#32cd32",
      },
    ]);
    if (str >= 70 && str <= 95) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff8c00",
        },
        {
          offset: 1,
          color: "#ff8c00",
        },
      ]);
    }
    if (str > 95) {
      color = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        {
          offset: 0,
          color: "#ff0000",
        },
        {
          offset: 1,
          color: "#ff0000",
        },
      ]);
    }
    return color;
  }
}

function loadMoniP() {
  axios({
    url: "http://192.168.20.130:9081/manager/v1.0/process",
    method: "get",
    params: {
      token: localStorage.getItem("Admin-Token"),
    },
  }).then((res) => {
    if (res.code === 1000) {
      this.processData = res.data;
    }
  });
}

function loadData() {
  store.dispatch("getDashboards");
  hardware.system = store.getters.hardware.server_version;
  hardware.setuptime = store.getters.hardware.system_time;
  stat.length = 0;
  stat.push(
    {
      icon: "fa-upload",
      title: "上行流量",
      total: store.getters.flows.total_flow_out_s + "Kbps",
      bgColor: "#ebcc6f",
    },
    {
      icon: "fa-download",
      title: "下行流量",
      total: store.getters.flows.total_flow_in_x + "Kbps",
      bgColor: "#3acaa9",
    },
    {
      icon: "fa-paper-plane",
      title: "发送总流量",
      total: store.getters.flows.total_bytes_sent,
      bgColor: "#67c4ed",
    },
    {
      icon: "fa-ethernet",
      title: "接收总流量",
      total: store.getters.flows.total_bytes_recv,
      bgColor: "#17c4fd",
    }
  );
  gaugeimg(
    "cpuTemp",
    "CPU温度",
    0,
    100,
    store.getters.charts.cpuTemp || 0,
    "℃"
  );
  gaugeimg(
    "cpuUsage",
    "CPU使用率",
    0,
    100,
    store.getters.charts.cpuUsage || 0,
    "%"
  );
  gaugeimg(
    "memUsage",
    "内存使用率",
    0,
    100,
    store.getters.charts.memUsage || 0,
    "%"
  );
  gaugeimg(
    "diskUsage",
    "磁盘使用率",
    0,
    100,
    store.getters.charts.diskUsage || 0,
    "%"
  );
}

function initCharts() {
  TempChart = echarts.init(cpuTemp.value);
  CpuChart = echarts.init(cpuUsage.value);
  MemChart = echarts.init(memUsage.value);
  DiskChart = echarts.init(diskUsage.value);
}

onUnmounted(() => {
  clearInterval(timer.value);
  ws && ws.close();
});
</script>
<style scoped>
.stat-info__item {
  display: flex;
  height: 50px;
  box-shadow: 2px 2px 5px #ccc;
  border: 1rem;
  border-radius: 1.5rem;
}
.stat-info__icon {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 58px;
  color: white;
  border: 1rem;
  border-radius: 1.5rem;
}
.stat-info__detail {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.stat-info__total {
  color: #0085d0;
  font-size: 18px;
  font-weight: bold;
}
.stat-info__title {
  color: #666;
  font-size: 12px;
  padding-top: 8px;
}
#dash {
  overflow-y: auto;
  height: 800px;
}
a:hover {
  background: linear-gradient(to left, #4650dd, #1d26a0) !important;
  color: #dee2e6;
}
.fa-upload {
  color: salmon;
}

.fa-download {
  color: green;
}

.fa-paper-plane {
  opacity: 1;
}

.fa-ethernet {
  color: rgb(59, 91, 152);
}
</style>