<script setup>
import { ref, h } from "vue";
import { Message, Modal } from "@arco-design/web-vue";
import { IconFaceSmileFill } from "@arco-design/web-vue/es/icon";
import { use, registerMap } from "echarts/core";
import VChart from "vue-echarts";
import { CanvasRenderer } from "echarts/renderers";
import { MapChart, ScatterChart, EffectScatterChart } from "echarts/charts";
import {
  TitleComponent,
  TooltipComponent,
  LegendComponent,
  GridComponent,
  VisualMapComponent,
} from "echarts/components";

use([
  CanvasRenderer,
  TitleComponent,
  TooltipComponent,
  LegendComponent,
  GridComponent,
  VisualMapComponent,
  MapChart,
  ScatterChart,
  EffectScatterChart,
]);
const myChart = ref();
const mapOption = ref();
const mapList = ref([
  {
    name: "中国",
    mapName: "100000_full",
  },
]); // 记录地图

const loading = ref(false);
const loadingOptions = {
  text: "加载中...",
  color: "#165dff",
  textColor: "#165dff",
  maskColor: "rgba(0, 0, 0, 0.04)",
};

const getMapJson = async (mapName) => {
  const url = `https://geo.datav.aliyun.com/areas_v3/bound/${mapName}.json`;
  const mapJson = await fetch(url).then((res) => res.json());
  return mapJson;
};

const setOptions = (mapName, mapData) => {
  return {
    tooltip: {
      show: false,
    },
    visualMap: {
      show: false,
      inRange: {
        color: ["#72deb1"],
      },
    },
    geo: {
      map: mapName,
      roam: false,
      select: false,
      zoom: 1.11,
      label: {
        show: false,
      },
      emphasis: {
        itemStyle: {
          areaColor: "#19cb81",
          borderColor: "#19cb81",
          borderWidth: 1,
        },
        label: {
          fontSize: 14,
          color: "#000",
        },
      },
      itemStyle: {
        borderColor: "#fafafa",
        borderWidth: 0.4,
      },
    },
    series: [
      // 数据
      {
        type: "map",
        map: mapName,
        roam: true,
        geoIndex: 0,
        select: false,
        data: mapData,
      },
    ],
  };
};

const renderMapECharts = async (mapName, random = false) => {
  const mapJson = await getMapJson(mapName);
  registerMap(mapName, mapJson);
  const mapData = mapJson.features.map((item) => {
    const tempValue = item.properties.center
      ? [...item.properties.center]
      : item.properties.center;
    return {
      name: item.properties.name,
      value: tempValue, // 中心点经纬度
      adcode: item.properties.adcode, // 区域编码
      level: item.properties.level, // 层级
    };
  });
  loading.value = false;
  mapOption.value = setOptions(mapName, mapData);
  if (random) {
    if (mapData && mapData.length > 1) {
      setTimeout(() => {
        statBtn();
      }, 2000);
    } else {
      setTimeout(() => {
        const content = mapList.value.map((item) => item.name).join("") + '  期待你的到来！';
        Modal.info({
          title: "冲冲冲",
          content
        });
        isDisabled.value = false
      }, 1500);
    }
  }
};

renderMapECharts("100000_full"); // 初始化绘制中国地图

// 点击下砖
const handleClick = (param, random = false) => {
  loading.value = true;
  // 只有点击地图才触发
  if (param.seriesType !== "map") return;
  const { adcode, level, name } = param.data;
  const mapName = level === "district" ? adcode : adcode + "_full";
  // 防止最后一个层级被重复点击，返回上一级出错
  if (mapList.value[mapList.value.length - 1].mapName === mapName) {
    loading.value = false;
    return Message.error({
      content: "到底了，不能再点了！",
      icon: () => h(IconFaceSmileFill),
    });
  }
  mapList.value.push({
    name,
    mapName,
  });
  renderMapECharts(mapName, random);
};

const goMap = (item) => {
  if (item.mapName === mapList.value[mapList.value.length - 1].mapName) {
    return;
  }
  renderMapECharts(item.mapName);
  // 删除当前地图之后的地图
  const index = mapList.value.findIndex((i) => i.mapName === item.mapName);
  mapList.value.splice(index + 1);
};

const actinNum = ref(null);
const timer = ref(null);
const isDisabled = ref(false);
const statBtn = () => {
  isDisabled.value = true
  let dataLength = mapOption.value.series[0].data.length;
  if(dataLength === 1) {
    Message.error('只有一个区域，不能开启');
    isDisabled.value = false
    return
  }
  // 先取消高亮已经高亮的
  timer.value = setInterval(() => {
    if (actinNum.value !== null) {
      myChart.value.chart.dispatchAction({
        type: "downplay",
        seriesIndex: 0,
        dataIndex: actinNum.value,
      });
    }

    actinNum.value = getRandomInt(dataLength);
    myChart.value.chart.dispatchAction({
      type: "highlight",
      seriesIndex: 0,
      dataIndex: actinNum.value,
    });
  }, 100);

  setTimeout(() => {
    outBtn();
  }, 4000);
};

const outBtn = () => {
  clearInterval(timer.value);
  timer.value = null;
  // 获取当前高亮的区域信息，并调用handleClick方法
  setTimeout(() => {
    const data = mapOption.value.series[0].data[actinNum.value];
    if (data) {
      Message.success(data.name);
      handleClick(
        {
          seriesType: "map",
          data,
        },
        true
      );
    }
    actinNum.value = null;
  }, 500);

};

function getRandomInt(max = 33) {
  return Math.floor(Math.random() * (max + 1));
}
</script>

<template>
  <div style="height: 100vh; width: 100vw; background-color: #eceff7">
    <div class="btn-box">
      <a-button type="primary" :disabled="isDisabled" @click="statBtn">开启</a-button>
      <!-- <a-button type="primary" @click="outBtn" v-else>停止</a-button> -->
      <div style="display: inline-block; margin-left: 50px">
        <a-breadcrumb>
          <a-breadcrumb-item v-for="(item, index) in mapList" @click="goMap(item)" :key="index">{{ item.name
            }}</a-breadcrumb-item>
        </a-breadcrumb>
      </div>
    </div>
    <v-chart ref="myChart" :loadingOptions="loadingOptions" :loading="loading" :option="mapOption" autoresize
      @click="handleClick" />
  </div>
</template>

<style scoped>
.btn-box {
  position: fixed;
  top: 20px;
  left: 50px;
  z-index: 999;
}

.item-ctlt {
  color: #000;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
}

.item-ctlt:hover {
  color: #19cb81;
}
</style>
