<template>
  <div class="flex flex-row justify-center w-full">
    <div class="mx-8 max-w-none w-full lg:w-2/3 flex flex-col">
      <!-- Loading state -->
      <div v-if="loading" class="flex justify-center items-center py-16">
        <span class="loading loading-spinner loading-lg"></span>
        <span class="ml-4 text-xl">{{ t('loading') }}</span>
      </div>

      <!-- Room details -->
      <div v-else-if="roomDetail && !error" class="flex flex-col gap-4 mt-4">
        <!-- Room info card -->
        <div class="card bg-base-100 shadow-xl p-6" :class="roomDetail.isLocked ? 'bg-error/20' : 'bg-success/20'">
          <div class="flex items-center gap-4">
            <div class="avatar">
              <div class="w-16 rounded-xl">
                <img 
                  v-if="hostUser?.avatar" 
                  :src="fileToURL(hostUser.avatar)" 
                  :alt="hostUser.name || `用户${roomDetail.host}`" 
                  class="w-full h-full object-cover"
                >
                <i v-else class="fa-solid fa-user w-full h-full flex items-center justify-center bg-base-300 p-4"></i>
              </div>
            </div>
            <div class="flex-1">
              <h1 class="text-3xl font-bold">{{ roomDetail.roomId }}</h1>
              <div class="flex flex-col gap-2 text-base-content/70 mt-2">
                <div class="flex items-center gap-2">
                  <i class="fa-solid fa-user"></i>
                  <span>{{ t('room-owner') }}: {{ hostUser?.name || `用户${roomDetail.host}` }}</span>
                </div>
                <div class="flex items-center gap-2">
                  <i class="fa-solid fa-users"></i>
                  <span>{{ roomDetail.players ? roomDetail.players.length : 0 }} {{ t('players') }}</span>
                </div>
                <div class="flex items-center gap-2">
                  <i class="fa-solid" :class="roomDetail.isLocked ? 'fa-lock' : 'fa-lock-open'"></i>
                  <span>{{ roomDetail.isLocked ? t('locked') : t('unlocked') }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Room players -->
        <div class="card bg-base-100 shadow-xl p-6">
          <h2 class="text-2xl font-bold mb-4">{{ t('room-players') }}</h2>
          <div v-if="!roomDetail.players || roomDetail.players.length === 0" class="text-center py-8 text-base-content/50">
            {{ t('no-players') }}
          </div>
          <div v-else class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
            <div 
              v-for="userId in roomDetail.players" 
              :key="userId"
              class="card bg-base-200 shadow-md p-4 cursor-pointer hover:shadow-lg transition-shadow"
              @click="goToUserDetail(userId)"
            >
              <div class="flex items-center gap-3">
                <div class="avatar">
                  <div class="w-10 rounded-xl">
                    <img 
                      v-if="userMap[userId]?.avatar" 
                      :src="fileToURL(userMap[userId].avatar)" 
                      :alt="userMap[userId].name || `用户${userId}`" 
                      class="w-full h-full object-cover"
                    >
                    <i v-else class="fa-solid fa-user w-full h-full flex items-center justify-center bg-base-300 p-2"></i>
                  </div>
                </div>
                <div>
                  <div class="font-medium">{{ userMap[userId]?.name || `用户${userId}` }}</div>
                  <div class="text-sm text-base-content/70">
                    {{ roomDetail.readyInfo && roomDetail.readyInfo[userId] ? t('ready') : t('not-ready') }}
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Room settings -->
        <div class="card bg-base-100 shadow-xl p-6">
          <h2 class="text-2xl font-bold mb-4">{{ t('room-settings') }}</h2>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="card bg-base-200 p-4">
              <h3 class="text-lg font-semibold mb-2">{{ t('room-type') }}</h3>
              <p>{{ getRoomType(roomDetail.type) }}</p>
            </div>
            <div class="card bg-base-200 p-4">
              <h3 class="text-lg font-semibold mb-2">{{ t('room-state') }}</h3>
              <p>{{ getRoomState(roomDetail.state) }}</p>
            </div>
          </div>
        </div>

        <!-- Selected charts -->
        <div class="card bg-base-100 shadow-xl p-6" v-if="roomDetail.selectedCharts && roomDetail.selectedCharts.length > 0">
          <h2 class="text-2xl font-bold mb-4">{{ t('selected-charts') }}</h2>
          <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
            <div v-for="chartId in roomDetail.selectedCharts" :key="'chart-' + chartId">
              <router-link 
                v-if="chartMap[chartId]"
                :to="{ name: 'chart', params: { id: chartId } }"
                class="group card relative image-full bg-base-100 shadow-xl aspect-[8/5] illustration chart-card min-h-0 min-w-0"
              >
                <figure>
                  <img class="w-full aspect-[8/5]" :src="fileToURL(chartMap[chartId].illustration) + '.thumbnail'" v-if="chartMap[chartId]?.illustration" />
                  <div v-else class="w-full aspect-[8/5] bg-base-200 flex items-center justify-center">
                    <i class="fa-solid fa-music text-4xl text-base-content/50"></i>
                  </div>
                </figure>
                <div class="absolute right-0 badge badge-primary z-[11] m-2 text-lg p-3">
                  {{ chartMap[chartId]?.level || '0' }}
                </div>
                <div class="card-body flex-col justify-end p-4 gap-0 min-h-0 min-w-0">
                  <p class="flex-none text-sm">{{ chartMap[chartId]?.composer || `ID: ${chartId}` }}</p>
                  <h2 class="card-title group-hover:link truncate">
                    {{ chartMap[chartId]?.name || `谱面${chartId}` }}
                  </h2>
                </div>
              </router-link>
              <div 
                v-else
                class="group card relative image-full bg-base-100 shadow-xl aspect-[8/5] illustration chart-card min-h-0 min-w-0 opacity-50"
              >
                <figure>
                  <div class="w-full aspect-[8/5] bg-base-200 flex items-center justify-center">
                    <i class="fa-solid fa-music text-4xl text-base-content/50"></i>
                  </div>
                </figure>
                <div class="card-body flex-col justify-end p-4 gap-0 min-h-0 min-w-0">
                  <p class="flex-none text-sm">{{ t('loading-chart') }}</p>
                  <h2 class="card-title truncate">
                    {{ t('selected-charts') }} {{ chartId }}
                  </h2>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Error state -->
      <div v-else-if="error" class="flex justify-center items-center py-16">
        <div class="text-center">
          <i class="fa-solid fa-exclamation-triangle text-4xl text-error mb-4"></i>
          <h2 class="text-2xl font-bold">{{ error === t('room-disbanded') ? t('room-disbanded') : t('loading-failed') }}</h2>
          <p class="text-lg mt-2">{{ error }}</p>
          <button v-if="error !== t('room-disbanded')" @click="fetchRoomDetail" class="btn btn-primary mt-4">重试</button>
          <button v-else @click="goToServerDetail" class="btn btn-primary mt-4">返回服务器</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import type { User, Chart } from '@/model';
import { MPAPI_BASE, useFetchApi, fileToURL } from '@/common';

const { t } = useI18n();
const route = useRoute();
const router = useRouter();
const fetchApi = useFetchApi();

const serverName = route.params.serverName as string;
const roomId = route.params.roomId as string;
const roomDetail = ref<any>(null);
const hostUser = ref<User | null>(null);
const userMap = ref<Record<number, User>>({});
const chartMap = ref<Record<number, Chart>>({});
const loading = ref(true);
const error = ref<string | null>(null);

// 获取房间详情
async function fetchRoomDetail() {
  loading.value = true;
  error.value = null;

  try {
    // 获取房间详细信息
    const roomResponse = await fetch(`${MPAPI_BASE}/api/Room/${serverName}/${roomId}`);
    const roomData = await roomResponse.json();

    if (roomData.success) {
      roomDetail.value = roomData.room;
      console.log('Fetched room data:', roomDetail.value); // 调试日志
      
      // 检查房间是否已解散（Players数组为空）
      if (!roomDetail.value.players || roomDetail.value.players.length === 0) {
        console.log('Room has been disbanded on initial load'); // 调试日志
        error.value = t('room-disbanded');
        loading.value = false; // 确保加载状态结束
        return;
      }

      // 获取房主信息
      if (roomDetail.value.host) {
        try {
          const hostResponse = await fetchApi(`/user/${roomDetail.value.host}`);
          hostUser.value = hostResponse as User;
        } catch (err) {
          console.error(`Failed to fetch host user ${roomDetail.value.host}:`, err);
        }
      }

      // 获取房间中所有玩家的信息
      if (roomDetail.value.players && Array.isArray(roomDetail.value.players)) {
        for (const userId of roomDetail.value.players) {
          try {
            const userResponse = await fetchApi(`/user/${userId}`);
            userMap.value[userId] = userResponse as User;
          } catch (err) {
            console.error(`Failed to fetch user ${userId}:`, err);
          }
        }
      }
      
      // 获取房间中所有选中谱面的信息
      if (roomDetail.value.selectedCharts && Array.isArray(roomDetail.value.selectedCharts)) {
        for (const chartId of roomDetail.value.selectedCharts) {
          try {
            const chartResponse = await fetchApi(`/chart/${chartId}`);
            chartMap.value[chartId] = chartResponse as Chart;
          } catch (err) {
            console.error(`Failed to fetch chart ${chartId}:`, err);
          }
        }
      }
    } else {
      error.value = '无法获取房间信息';
    }
  } catch (err) {
    console.error('Failed to fetch room detail:', err);
    error.value = '获取房间详情时发生错误';
  } finally {
    loading.value = false;
  }
}

// 获取房间类型描述
function getRoomType(type: number): string {
  if (type === 0) return t('normal');
  if (type === 1) return t('cycle');
  if (type === 2) return t('vote');
  return `${t('room-type')} ${type}`;
}

// 获取房间状态描述
function getRoomState(state: number): string {
  if (state === 0) return t('waiting-chart-selection');
  if (state === 1) return t('waiting-user-ready');
  if (state === 2) return t('playing');
  return `${t('room-state')} ${state}`;
}

let ws: WebSocket | null = null;

// 连接到房间WebSocket
function connectToRoomWs() {
  // 关闭之前的连接
  if (ws) {
    ws.close();
  }
  
  try {
    // 使用环境变量或默认值构建WebSocket URL
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const apiBase = MPAPI_BASE.replace(/^https?:\/\//, ''); // 移除协议部分
    const wsUrl = `${protocol}//${apiBase}/api/ws/${serverName}/${roomId}`;
    ws = new WebSocket(wsUrl);
    
    ws.onopen = () => {
      console.log('Connected to room WebSocket');
    };
    
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        console.log('Received WebSocket message:', data); // 调试日志
        
        if (data.type === 'roomChanged' && data.room) {
          // 更新房间信息
          updateRoomInfo(data.room);
        }
      } catch (e) {
        console.error('Error parsing WebSocket message:', e);
      }
    };
    
    ws.onclose = () => {
      console.log('Disconnected from room WebSocket');
      // 尝试重连（可选）
      setTimeout(() => {
        if (document.visibilityState === 'visible') {
          connectToRoomWs();
        }
      }, 3000);
    };
    
    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  } catch (e) {
    console.error('Failed to create WebSocket connection:', e);
  }
}

// 更新房间信息
function updateRoomInfo(newRoomData: any) {
  console.log('Updating room info:', newRoomData); // 调试日志
  if (roomDetail.value) {
    // 检查房间是否已解散（Players数组为空）
    if (!newRoomData.Players || newRoomData.Players.length === 0) {
      console.log('Room has been disbanded'); // 调试日志
      // 房间已解散，显示提示并重定向
      error.value = t('room-disbanded');
      loading.value = false; // 确保加载状态结束
      // 可以选择重定向到服务器页面或其他适当的处理
      // router.push({ name: 'mpserver-detail', params: { id: serverName } });
      return;
    }
    
    // 更新房间的基本属性
    roomDetail.value.roomId = newRoomData.RoomId;
    roomDetail.value.host = newRoomData.Host;
    roomDetail.value.players = newRoomData.Players;
    roomDetail.value.isLocked = newRoomData.IsLocked;
    roomDetail.value.state = newRoomData.State;
    roomDetail.value.type = newRoomData.Type;
    roomDetail.value.selectedCharts = newRoomData.SelectedCharts;
    roomDetail.value.readyInfo = newRoomData.ReadyInfo;
    roomDetail.value.monitors = newRoomData.Monitors;
    
    // 更新房主信息
    if (roomDetail.value.host) {
      fetchUser(roomDetail.value.host).then(user => {
        if (user) {
          hostUser.value = user;
        }
      });
    }
    
    // 更新玩家信息
    if (roomDetail.value.players && Array.isArray(roomDetail.value.players)) {
      console.log('Updating player list:', roomDetail.value.players); // 调试日志
      // 清空旧的用户映射
      for (const userId of Object.keys(userMap.value)) {
        delete userMap.value[Number(userId)];
      }
      
      for (const userId of roomDetail.value.players) {
        fetchUser(userId).then(user => {
          if (user) {
            userMap.value[userId] = user;
            console.log('Updated user info:', userId, user); // 调试日志
          }
        });
      }
    }
    
    // 更新谱面信息
    if (roomDetail.value.selectedCharts && Array.isArray(roomDetail.value.selectedCharts)) {
      for (const chartId of roomDetail.value.selectedCharts) {
        fetchChart(chartId).then(chart => {
          if (chart) {
            chartMap.value[chartId] = chart;
          }
        });
      }
    }
  }
}

// 获取用户信息
async function fetchUser(userId: number): Promise<User | null> {
  try {
    const user = await fetchApi(`/user/${userId}`);
    return user as User;
  } catch (error) {
    console.error(`Failed to fetch user ${userId}:`, error);
    return null;
  }
}

// 获取谱面信息
async function fetchChart(chartId: number): Promise<Chart | null> {
  try {
    const chart = await fetchApi(`/chart/${chartId}`);
    return chart as Chart;
  } catch (error) {
    console.error(`Failed to fetch chart ${chartId}:`, error);
    return null;
  }
}

onMounted(() => {
  fetchRoomDetail();
  // 连接WebSocket以接收实时更新
  connectToRoomWs();
});

onUnmounted(() => {
  // 组件卸载时关闭WebSocket连接
  if (ws) {
    ws.close();
  }
});

// 跳转到用户详情页
function goToUserDetail(userId: number) {
  router.push({ name: 'user', params: { id: userId } });
}

// 跳转到谱面详情页
function goToChartDetail(chartId: number) {
  router.push({ name: 'chart', params: { id: chartId } });
}

// 跳转到服务器详情页
function goToServerDetail() {
  router.push({ name: 'mpserver-detail', params: { id: serverName } });
}
</script>

<i18n>
en:
  loading: Loading room details...
  room-players: Room Players
  no-players: No players in the room
  room-settings: Room Settings
  room-type: Room Type
  room-state: Room State
  selected-charts: Selected Charts
  loading-chart: Loading...
  room-owner: Room Owner
  players: Players
  locked: Locked
  unlocked: Unlocked
  ready: Ready
  not-ready: Not Ready
  waiting-chart-selection: Waiting for chart selection
  waiting-user-ready: Waiting for user ready
  playing: Playing
  normal: Normal
  cycle: Cycle
  vote: Vote
  loading-failed: Loading failed
  room-disbanded: Room has been disbanded

zh-CN:
  loading: 加载房间详情中...
  room-players: 房间玩家
  no-players: 房间中暂无玩家
  room-settings: 房间设置
  room-type: 房间类型
  room-state: 房间状态
  selected-charts: 已选谱面
  loading-chart: 加载中...
  room-owner: 房主
  players: 玩家
  locked: 已锁定
  unlocked: 未锁定
  ready: 已准备
  not-ready: 未准备
  waiting-chart-selection: 等待谱面选取
  waiting-user-ready: 等待用户准备
  playing: 正在游玩
  normal: 普通
  cycle: 循环
  vote: 投票
  loading-failed: 加载失败
  room-disbanded: 房间已解散
</i18n>