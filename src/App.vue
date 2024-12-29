<script setup>
import {computed, onMounted, provide, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import axios from "axios";
import { Message } from "@arco-design/web-vue";
import StatsCard from "@/components/StatsCard.vue";
import {formatBytes, formatTimeStamp, formatUptime, calculateRemainingDays} from '@/utils/utils'

const socketURL = ref('')
const apiURL = ref('')

const theme = window.localStorage.getItem('theme') || 'light'
const dark = ref(theme !== 'light')

const handleChangeDark = () => {
  dark.value = !dark.value

  if (dark.value) {
    window.localStorage.setItem('theme','dark')
    document.body.setAttribute('arco-theme', 'dark')
  } else {
    // 恢复亮色主题
    window.localStorage.setItem('theme','light')
    document.body.removeAttribute('arco-theme');
   }
}

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }

  return data.value.filter(item => item.Host.Name.slice(0, 2) === selectArea.value)
})

const hosts = computed(() => {
  if (type.value === 'all') {
    return host.value
  } else if (type.value === 'online') {
    return host.value.filter(item => item.status)
  } else {
    return host.value.filter(item => !item.status)
  }
})

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetOutSpeed
    bandwidth_down += item.State.NetInSpeed
    traffic_up += item.State.NetOutTransfer
    traffic_down += item.State.NetInTransfer
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down,
    traffic_up: traffic_up,
    traffic_down: traffic_down
  }
})

let socket = null

let nowtime = (Math.floor(Date.now() / 1000))

const fetchConfig = async () => {
  try {
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
  } catch (e) {
    Message.error('获取配置失败')
  }
}

const initScoket = async () => {
  await fetchConfig()

  socket = new WebSocket(socketURL.value);  // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function(event) {
    try {
      const message = event.data;
      const res = JSON.parse(message.replace('data: ', ''))
      if (res != null) {
        area.value = Array.from(new Set(res.map(item => item.Host.Name.slice(0, 2))))
      }
      data.value = res.map((host) => {
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length == 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetOutSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetInSpeed])

        return {
          ...host,
          status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
        }
      })

      setTimeout(() => sendPing(), 1000)

    } catch (error) {
      console.error('解析 WebSocket 消息时出错:', error);
    }
  };

  socket.onopen = function () {
    sendPing()
  }

  socket.onclose = function () {
    Message.warning('WebSocket已断连，正在重连中...')

    initScoket()
  }
}

const sendPing = () => {
  nowtime = (Math.floor(Date.now() / 1000))
  socket.send('ping')
}

onMounted(async() => {
  if (dark.value) {
    document.body.setAttribute('arco-theme', 'dark')
  }

  await initScoket()
  handleFetchHostInfo()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'danger';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

// 删除主机
const deleteVisible = ref(false)
const authSecret = ref('')
const deleteHostName = ref('')

const handleShowDelete = (name) => {
  authSecret.value =  window.localStorage.getItem('auth_secret') || ''
  deleteHostName.value = name
  deleteVisible.value = true
}

const handleDeleteHost = async () => {
  try {
    await axios.post(apiURL.value + '/delete', {
      "auth_secret": authSecret.value,
      "name": deleteHostName.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('删除成功')

    deleteVisible.value = false
  } catch (e) {
    Message.error('删除失败，管理密钥错误')
  }
}

const handleClose = () => {
  deleteVisible.value = false
}

const hostInfo = ref({})

const handleFetchHostInfo = async () => {
  try {
    const res = await axios.get(apiURL.value + '/info')
    res.data.forEach((item) => {
      hostInfo.value[item.name] = item
    })
  } catch (e) {
    // Message.error('删除失败，管理密钥错误')
  }
}

const handleChangeType = (value) => {
  type.value = value
}

const editHostName = ref('')
const duetime = ref(0)
const buy_url = ref('')
const seller = ref('')
const price = ref('')

const editVisible = ref(false)

const handleShowEdit = (name) => {
  if (!hostInfo.value[name]) {
    editHostName.value = name
    duetime.value = 0
    buy_url.value = ''
    seller.value = ''
    price.value = ''
    editVisible.value = true
    authSecret.value = window.localStorage.getItem('auth_secret') || ''
    return
  }

  editHostName.value = name
  duetime.value = hostInfo.value[name].due_time
  buy_url.value = hostInfo.value[name].buy_url
  seller.value = hostInfo.value[name].seller
  price.value = hostInfo.value[name].price
  editVisible.value = true
  authSecret.value = window.localStorage.getItem('auth_secret') || ''

}

const handleEditHost = async () => {
  try {
    await axios.post(apiURL.value + '/info', {
      "name": editHostName.value,
      "due_time": new Date(duetime.value).getTime(),
      "buy_url": buy_url.value,
      "seller": seller.value,
      "auth_secret": authSecret.value,
      "price": price.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success('更新成功')

    handleFetchHostInfo()

    editVisible.value = false
  } catch (e) {
    Message.error('更新失败，管理密钥错误')
  }

}

const handleEditClose = () => {
  editVisible.value = false
}

provide('handleChangeType', handleChangeType)

</script>

<template>
  <div class="max-container">
    <div class="header">
      <a class="logo" href="#">
        <svg class="arco-icon" viewBox="0 0 48 48" fill="currentColor">
          <path data-v-2ee6cb6b="" fill-rule="evenodd" clip-rule="evenodd" d="M42.919 11.923L25 1.577a2 2 0 00-2 0L5.081 11.923a2 2 0 00-1 1.732v20.69a2 2 0 001 1.732L23 46.423a2 2 0 002 0l17.919-10.346a2 2 0 001-1.732v-20.69a2 2 0 00-1-1.732zM30.556 9.525L38.5 14 24 23l-13.808-8.668L17.5 10l6.5 4 6.556-4.475zM22 40.441V26.286L8 17.358v7.928l8 5.464v6.227l6 3.464zm10-3.464l-6 3.464V26.286l14-8.928v8.928l-8 5.464v5.227z" fill="currentColor"></path>
        </svg>
        <span>Akile Monitor</span>
        <small style="font-weight: 400;opacity: .8"> ｜ 全球节点监控</small>
      </a>
      <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">
        <template #icon>
          <icon-sun-fill v-if="!dark" />
          <icon-moon-fill v-else />
        </template>
      </a-button>
    </div>
    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        全部地区
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>
    <StatsCard :type="type" :stats="stats" @handleChangeType="handleChangeType" />
    <div class="monitor-card">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in hosts" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status  ? '在线' : '离线'}}</span>
            <span style="margin-left: 6px;">{{formatUptime(item.State.Uptime)}}</span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">系统</div>
          <div class="monitor-item-value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">内存使用情况</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">网络速度（IN|OUT）</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">负载平均值(1|5|15)</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <!-- <div class="uptime" style="width: 120px;">
          <div class="monitor-item-title">剩余时间</div>
          <div class="monitor-item-value">{{hostInfo[item.Host.Name] ? calculateRemainingDays(hostInfo[item.Host.Name].due_time) : '-'}}</div>
        </div> -->
        <div class="uptime">
          <div class="monitor-item-title">上报时间</div>
          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>
        </div>
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">主机名</div>
                  <div class="value">{{item.Host.Name}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">地区</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Name.slice(0, 2).toUpperCase()}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">系统</div>
                  <div class="value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">架构</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟化</div>
                  <div class="value">{{item.Host.Virtualization || '-'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU占用</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">内存使用情况</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatBytes(item.State.MemUsed)}} / {{formatBytes(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟内存(Swap)</div>
                  <div class="value">{{formatBytes(item.State.SwapUsed)}} / {{formatBytes(item.Host.SwapTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">网络速度（IN|OUT）</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">负载平均值(1|5|15)</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">流量使用↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetOutTransfer)}} | {{formatBytes(item.State.NetInTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">开机时间</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">上报时间</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].seller">
                  <div class="name">商家名称</div>
                  <div class="value">{{hostInfo[item.Host.Name].seller}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].price">
                  <div class="name">主机价格</div>
                  <div class="value">{{hostInfo[item.Host.Name].price}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].due_time">
                  <div class="name">到期时间</div>
                  <div class="value">{{moment(hostInfo[item.Host.Name].due_time).format('YYYY-MM-DD')}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].buy_url">
                  <div class="name">购买链接</div>
                  <div class="value">
                    <a style="color: #0077ff" :href="hostInfo[item.Host.Name].buy_url" target="_blank" @click.stop="() => {}">{{hostInfo[item.Host.Name].buy_url}}</a>
                  </div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <CPU ref="cpuRef" style="margin-bottom: 20px;" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" style="margin-bottom: 20px;" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
        <div class="edit-btn" @click.stop="handleShowEdit(item.Host.Name)">
          <icon-edit />
        </div>
        <div class="delete-btn" @click.stop="handleShowDelete(item.Host.Name)">
          <icon-delete />
        </div>
      </div>
    </div>
    <a-modal v-model:visible="deleteVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>删除主机</span>
        <a-button @click="handleClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-input-password v-model="authSecret" placeholder="请输入管理密钥"></a-input-password>
        <div class="tips">提示：删除后无法恢复，请确定后再删除操作</div>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" status="danger" :long="true" @click="handleDeleteHost">确认删除</a-button>
      </div>
    </a-modal>
    <a-modal v-model:visible="editVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>编辑主机信息</span>
        <a-button @click="handleEditClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-date-picker v-model="duetime" placeholder="请选择到期时间" style="margin-bottom: 10px;width: 100%;"></a-date-picker>
        <a-input v-model="seller" placeholder="请输入卖家" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="price" placeholder="请输入价格" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="buy_url" placeholder="请输入购买链接" style="margin-bottom: 10px;"></a-input>
        <a-input-password v-model="authSecret" placeholder="请输入管理密钥"></a-input-password>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" :long="true" @click="handleEditHost">更新信息</a-button>
      </div>
    </a-modal>
    <div class="footer" style="margin-top: 30px">代码开源在 <a href="https://github.com/akile-network/akile_monitor">GitHub v0.0.2</a></div>
    <div class="footer" style="margin-bottom: 30px">Copyright © 2023-{{new Date().getFullYear()}} Akile LTD.</div>
  </div>
</template>

<style lang="scss">
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

/* 背景图片 */
body::before {
    content: "";
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
    background: url(https://api.imlazy.ink/img) center/cover no-repeat;
    filter: blur(0px);
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    border: 1px solid #eeeeee!important;
    background-color: #ffffff!important;
    color: #333333!important;
  }
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      background: #e7e7e790;
      backdrop-filter: blur(8px);

      .delete-btn,
      .edit-btn {
        display: none!important;
      }

      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e5e5e5b0;
      backdrop-filter: blur(10px);

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px!important;
      background: rgba(22, 131, 255, 0.13)!important;

      &:hover {
        background: rgba(22, 131, 255, 0.19)!important;
      }

      .arco-icon {
        color: #1673ff!important;
      }
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      background: #ff161620;
      border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: .15s background-color ease-in-out;

      &:hover {
        background: #ff161630;
      }

      .arco-icon {
        color: #ff1616;
      }
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;
        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: .6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .uptime {
      display: inline-block;
      vertical-align: middle;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 0%;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .arco-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22,93,255)!important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  line-height: 22px;
  text-align: center;
  color: #565656;

  a {
    color: rgba(var(--primary-6));
  }
}

.akile-modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 20px;
}

.akile-modal-content {
  margin-bottom: 20px;
  .tips {
    font-size: 12px;
    color: #333333;
    margin-top: 10px;
  }
}

body[arco-theme='dark'] {
  background-color: #111111;

  .arco-modal {
    background-color: #0e0e0e;
    border: 1px solid rgba(255,255,255,0.05);
  }

  .header {
    .logo {
      span,
      small {
        color: #ffffff;
      }
    }

    .theme-btn {
      border: 1px solid #333333!important;
      background-color: #000000!important;
      color: #ffffff!important;
    }
  }

  .area-tabs {
    .area-tab-item {
      border-color: #333333;
      background: #000000;
      color: #ffffff;
      box-shadow: none;

      &.is-active {
        background: #005fe705;
        color: #005fe7;
        border: 1px solid #005fe7;
      }
    }
  }

  .footer {
    color: #ffffffAA;
  }

  .monitor-card {
    .monitor-item {
      border: 1px solid rgb(255 255 255 / 16%);
      box-shadow: none;
      background-color: #000000;
      color: #ffffff;

      &:hover {
        background-color: #101010;
      }

      .detail {
        border-color: #333333AA;
        .detail-item-list {
          .detail-item {
            .name {
              color: #999999;
            }
          }
        }
      }
    }
  }

}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}
</style>
