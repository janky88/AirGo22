<template>
  <div>
    <el-row :gutter="15">
      <el-col :xs="24" :sm="24" :md="24" :lg="24" :xl="24" v-for="(v,k) in serverStatusData.data" :key="k">
        <div class="home-card-item">
          <el-card>
            <el-row :gutter="10" justify="space-around" align="middle">
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin: auto">
                <el-text style="color: #F9A43D;font-size: 15px;font-weight: bolder">{{ v.name }}</el-text>
              </el-col>
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin: auto">
                <el-icon v-if="v.status" color="#90ee90" :size="20">
                  <SuccessFilled/>
                </el-icon>
                <el-icon v-else color="#ff4d00" :size="20">
                  <CircleCloseFilled/>
                </el-icon>
              </el-col>
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin-top: 10px;margin-bottom: 10px">
                rate：{{ v.traffic_rate }}
              </el-col>
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin-top: 10px;margin-bottom: 10px">
                online：{{ v.user_amount }}
              </el-col>
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin: auto">
                <el-icon color="#409EFC">
                  <Top/>
                </el-icon>
                <span v-if="v.u < 1024">{{ v.u.toFixed(2) }}B/s</span>
                <span v-else-if="1024 < v.u < 1048576">{{ (v.u/1024).toFixed(2) }}kB/s</span>
                <span v-else-if="1048576 < v.u < 1073741824">{{ (v.u/1024/1024).toFixed(2) }}MB/s</span>
                <span v-else>{{ v.u }}B/s</span>
              </el-col>
              <el-col :xs="12" :sm="12" :md="4" :lg="4" :xl="4" style="margin: auto">
                <el-icon color="#409EFC">
                  <Bottom/>
                </el-icon>
                <span v-if="v.d < 1024">{{ v.d.toFixed(2) }}B/s</span>
                <span v-else-if="1024 < v.d < 1048576">{{ (v.d/1024).toFixed(2) }}kB/s</span>
                <span v-else-if="1048576 < v.d < 1073741824">{{ (v.d/1024/1024).toFixed(2) }}MB/s</span>
                <span v-else>{{ v.d }}B/s</span>
              </el-col>
            </el-row>

            <el-row :gutter="10">
              <el-col :xs="24" :sm="24" :md="8" :lg="8" :xl="8" style="margin-top: 5px;margin-bottom: 5px">
                <el-progress :text-inside="true" :stroke-width="20" :percentage="v.cpu.toFixed(0)" striped striped-flow :color="customColors">
                  <template #default="{ percentage }">
                    <span class="percentage-label">cpu：</span>
                    <span class="percentage-value">{{ percentage }}%</span>
                  </template>
                </el-progress>
              </el-col>
              <el-col :xs="24" :sm="24" :md="8" :lg="8" :xl="8" style="margin-top: 5px;margin-bottom: 5px">
                <el-progress :text-inside="true" :stroke-width="20" :percentage="v.mem.toFixed(0)" striped striped-flow :color="customColors">
                  <template #default="{ percentage }">
                    <span class="percentage-label">memory：</span>
                    <span class="percentage-value">{{ percentage }}%</span>
                  </template>
                </el-progress>
              </el-col>
              <el-col :xs="24" :sm="24" :md="8" :lg="8" :xl="8" style="margin-top: 5px;margin-bottom: 5px">
                <el-progress :text-inside="true" :stroke-width="20" :percentage="v.disk.toFixed(0)" striped striped-flow :color="customColors">
                  <template #default="{ percentage }">
                    <span class="percentage-label">disk：</span>
                    <span class="percentage-value">{{ percentage }}%</span>
                  </template>
                </el-progress>
              </el-col>
            </el-row>
          </el-card>
        </div>
      </el-col>
    </el-row>
  </div>

</template>

<script setup lang="ts">
import {onMounted, onUnmounted} from "vue";
import {useApiStore} from "/@/stores/apiStore";
import {storeToRefs} from "pinia";
import {useNodeStore} from "/@/stores/nodeStore";
import {Local} from "/@/utils/storage";

const nodeStore = useNodeStore()
const {serverStatusData} = storeToRefs(nodeStore)
const token = Local.get('token')
const apiStore = useApiStore()
const apiStoreData = storeToRefs(apiStore)
const customColors = [
  { color: '#5cb87a', percentage: 1 },
  { color: '#19befa', percentage: 40 },
  { color: '#f1d85f', percentage: 60 },
  { color: '#ef8b41', percentage: 70 },
  { color: '#f56c6c', percentage: 80 },
  { color: '#000000', percentage: 100 },
]


function getWsUrl(): string {
  let apiUrl: string = import.meta.env.VITE_API_URL
  if (apiUrl === '') {
    apiUrl = window.location.protocol + "//" + window.location.hostname + ":" + window.location.port
  }
  const url = apiUrl.slice(apiUrl.indexOf('//') + 2, apiUrl.length)
  const pre_url = apiUrl.slice(0, apiUrl.indexOf('//') + 2)
  // console.log(`pre_url:${pre_url} url:${url}`)
  if (pre_url === 'https://') {
    return "wss://" + url + apiStoreData.api.value.websocket_msg.path
  } else {
    return "ws://" + url + apiStoreData.api.value.websocket_msg.path
  }
}

let ws = new WebSocket(getWsUrl(), token);
let interval = null;//计时器
//监听是否连接成功
function initWS() {
  ws.onopen = function () {
    // console.log('ws连接成功,连接状态：' + ws.readyState);
    ws.send('{"type":1,"data":"hi"}');
    interval = setInterval(() => {
      ws.send('{"type":1,"data":"hi"}');
    }, 5000);
  }
//接收服务器发回的信息
  ws.onmessage = function (data) {
    // console.log('ws接收服务器发回的信息：' + ws.readyState);
    serverStatusData.value = JSON.parse(data.data)
    // console.log("JSON.parse:", JSON.parse(data.data))
  }
//监听连接关闭事件
  ws.onclose = function () {
    // console.log('ws连接关闭,连接状态：' + ws.readyState);
    clearInterval(interval)
    ws.close();
  }
//监听并处理error事件
  ws.onerror = function (error) {
    // console.log(error);
    ws.close();
    clearInterval(interval)
  }
}

onMounted(() => {
  initWS()
});
onUnmounted(() => {
  //完成通信后关闭WebSocket连接
  ws.close();
  clearInterval(interval)
});
</script>

<style scoped>
.home-card-item {
  font-size: 16px;
  width: 100%;
  height: 100%;
  border-radius: 4px;
  transition: all ease 0.3s;
  padding: 10px;
  overflow: hidden;
  background: var(--el-color-white);
  color: var(--el-text-color-primary);
  border: 1px solid var(--next-border-color-light);
}

.el-card {
  background-image: url("../../assets/bgc/3.png");
  background-repeat: no-repeat;
}
</style>