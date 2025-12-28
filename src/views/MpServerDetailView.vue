<i18n>
en:
  loading: Loading server details...
  online-users: Online Users
  rooms: Rooms
  no-users: No users online
  no-rooms: No rooms available
  room-placeholder: Room details coming soon...

zh-CN:
  loading: 加载服务器详情中...
  online-users: 在线用户
  rooms: 房间列表
  no-users: 暂无在线用户
  no-rooms: 暂无房间
  room-placeholder: 房间详情功能开发中...
</i18n>

<script setup lang="ts">
import { ref, onMounted, computed, onUnmounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import type { Server, User } from '@/model';
import { MPAPI_BASE, useFetchApi, fileToURL } from '@/common';
import OnlineUserCard from '../components/OnlineUserCard.vue';

const { t } = useI18n();
const route = useRoute();
const router = useRouter();
const fetchApi = useFetchApi();

const serverName = route.params.id as string; // 服务器名称作为唯一标识符
const server = ref<Server>();
const userIds = ref<number[]>([]);
const rooms = ref<any[]>([]);
const loading = ref(true);

const validRooms = computed(() => {
  return rooms.value.filter(room => room && room.roomId);
});

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

// 获取服务器详情
async function fetchServerDetail() {
  loading.value = true;

  try {
    // 获取服务器详细信息
    const detailsResponse = await fetch(`${MPAPI_BASE}/api/Server/${serverName}/details`);
    const detailsData = await detailsResponse.json();

    if (detailsData.success) {
      // 构造服务器信息
      server.value = {
        id: 0, // API 中未提供 id，使用 0 作为占位符
        name: serverName,
        address: detailsData.externalAddress || `${serverName}.phira-mp.com:12345`, // 使用API返回的外部地址
        roomCount: 0, // 房间数量将在后续 API 调用中获取
        maxUsers: detailsData.maxPlayers,
        currentUsers: detailsData.currentPlayers,
        uptime: detailsData.uptime, // API 现在直接返回秒数
      };

      // 获取房间列表
      const roomsResponse = await fetch(`${MPAPI_BASE}/api/Room/${serverName}`);
      const roomsData = await roomsResponse.json();

      if (roomsData.success) {
        server.value.roomCount = roomsData.roomCount;
        
        // 获取每个房间的详细信息
        // 根据API更正，rooms是一个字符串数组，每个字符串是一个房间ID
        const roomDetailsPromises = roomsData.rooms.map(async (roomId: string) => {
          try {
            const roomDetailResponse = await fetch(`${MPAPI_BASE}/api/Room/${serverName}/${roomId}`);
            const roomDetailData = await roomDetailResponse.json();
            
            if (roomDetailData.success) {
              const roomDetail = roomDetailData.room;
              // 获取房主的用户信息
              const hostUser = await fetchUser(roomDetail.host);
              
              return {
                ...roomDetail,
                hostName: hostUser ? hostUser.name : `用户${roomDetail.host}`,
                hostAvatar: hostUser ? hostUser.avatar : null
              };
            }
          } catch (error) {
            console.error(`Failed to fetch room details for room ${roomId}:`, error);
          }
          
          // 如果获取房间详情失败，返回基本房间信息
          return {
            roomId: roomId,
            host: 0, // 从详细API获取
            hostName: '未知用户', // 待实现：获取用户名
            players: [], // 从详细API获取
            isLocked: false, // 从详细API获取
          };
        });
        
        // 过滤掉无效的房间
        const allRoomDetails = await Promise.all(roomDetailsPromises);
        const newRooms = allRoomDetails.filter((room): room is any => room !== undefined && room !== null);
        rooms.value = newRooms;
        
        // 为每个房间建立WebSocket连接
        for (const room of newRooms) {
          connectToRoomWs(room.roomId);
        }
      }
      
      // 获取服务器中的玩家列表
      const playersResponse = await fetch(`${MPAPI_BASE}/api/Player/${serverName}`);
      const playersData = await playersResponse.json();
      
      if (playersData.success) {
        userIds.value = playersData.players;
      }
    }
  } catch (error) {
    console.error('Failed to fetch server details:', error);
  } finally {
    loading.value = false;
  }
}

let serverWs: WebSocket | null = null;
const roomWss: Record<string, WebSocket> = {};

// 连接到服务器房间列表WebSocket
function connectToServerWs() {
  // 关闭之前的连接
  if (serverWs) {
    serverWs.close();
  }
  
  try {
    // 使用环境变量或默认值构建WebSocket URL
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const apiBase = MPAPI_BASE.replace(/^https?:\/\//, ''); // 移除协议部分
    const wsUrl = `${protocol}//${apiBase}/api/ws/${serverName}/rooms`;
    serverWs = new WebSocket(wsUrl);
    
    serverWs.onopen = () => {
      console.log('Connected to server rooms WebSocket');
    };
    
    serverWs.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        console.log('Received server WebSocket message:', data); // 调试日志
        
        if (data.type === 'roomsChanged' && data.rooms) {
          // 重新获取房间列表
          refreshRoomsList();
        }
      } catch (e) {
        console.error('Error parsing WebSocket message:', e);
      }
    };
    
    serverWs.onclose = () => {
      console.log('Disconnected from server rooms WebSocket');
      // 尝试重连（可选）
      setTimeout(() => {
        if (document.visibilityState === 'visible') {
          connectToServerWs();
        }
      }, 3000);
    };
    
    serverWs.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  } catch (e) {
    console.error('Failed to create WebSocket connection:', e);
  }
}

// 连接到指定房间的WebSocket
function connectToRoomWs(roomId: string) {
  // 如果已存在连接，先关闭
  if (roomWss[roomId]) {
    roomWss[roomId].close();
  }
  
  try {
    // 使用环境变量或默认值构建WebSocket URL
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const apiBase = MPAPI_BASE.replace(/^https?:\/\//, ''); // 移除协议部分
    const wsUrl = `${protocol}//${apiBase}/api/ws/${serverName}/${roomId}`;
    roomWss[roomId] = new WebSocket(wsUrl);
    
    roomWss[roomId].onopen = () => {
      console.log(`Connected to room ${roomId} WebSocket`);
    };
    
    roomWss[roomId].onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        console.log(`Received room ${roomId} WebSocket message:`, data); // 调试日志
        
        if (data.type === 'roomChanged' && data.room) {
          // 更新特定房间的信息
          updateRoomInfo(roomId, data.room);
        }
      } catch (e) {
        console.error('Error parsing room WebSocket message:', e);
      }
    };
    
    roomWss[roomId].onclose = () => {
      console.log(`Disconnected from room ${roomId} WebSocket`);
      // 尝试重连（可选）
      setTimeout(() => {
        if (document.visibilityState === 'visible') {
          connectToRoomWs(roomId);
        }
      }, 3000);
    };
    
    roomWss[roomId].onerror = (error) => {
      console.error(`Room ${roomId} WebSocket error:`, error);
    };
  } catch (e) {
    console.error(`Failed to create room ${roomId} WebSocket connection:`, e);
  }
}

// 断开指定房间的WebSocket连接
function disconnectRoomWs(roomId: string) {
  if (roomWss[roomId]) {
    roomWss[roomId].close();
    delete roomWss[roomId];
  }
}

// 更新特定房间的信息
function updateRoomInfo(roomId: string, newRoomData: any) {
  const roomIndex = rooms.value.findIndex(r => r.roomId === roomId);
  if (roomIndex !== -1) {
    // 更新房间的基本属性
    rooms.value[roomIndex].roomId = newRoomData.RoomId;
    rooms.value[roomIndex].host = newRoomData.Host;
    rooms.value[roomIndex].players = newRoomData.Players;
    rooms.value[roomIndex].isLocked = newRoomData.IsLocked;
    rooms.value[roomIndex].state = newRoomData.State;
    rooms.value[roomIndex].type = newRoomData.Type;
    rooms.value[roomIndex].selectedCharts = newRoomData.SelectedCharts;
    rooms.value[roomIndex].readyInfo = newRoomData.ReadyInfo;
    rooms.value[roomIndex].monitors = newRoomData.Monitors;
    
    // 重新获取房主的用户信息
    if (rooms.value[roomIndex].host) {
      fetchUser(rooms.value[roomIndex].host).then(user => {
        if (user) {
          rooms.value[roomIndex].hostName = user.name;
          rooms.value[roomIndex].hostAvatar = user.avatar;
        }
      });
    }
  }
}

// 刷新房间列表
async function refreshRoomsList() {
  try {
    // 获取房间列表
    const roomsResponse = await fetch(`${MPAPI_BASE}/api/Room/${serverName}`);
    const roomsData = await roomsResponse.json();

    if (roomsData.success) {
      // 更新服务器房间数量
      if (server.value) {
        server.value.roomCount = roomsData.roomCount;
      }
      
      // 获取每个房间的详细信息
      // 根据API更正，rooms是一个字符串数组，每个字符串是一个房间ID
      const roomDetailsPromises = roomsData.rooms.map(async (roomId: string) => {
        try {
          const roomDetailResponse = await fetch(`${MPAPI_BASE}/api/Room/${serverName}/${roomId}`);
          const roomDetailData = await roomDetailResponse.json();
          
          if (roomDetailData.success) {
            const roomDetail = roomDetailData.room;
            // 获取房主的用户信息
            const hostUser = await fetchUser(roomDetail.host);
            
            return {
              ...roomDetail,
              hostName: hostUser ? hostUser.name : `用户${roomDetail.host}`,
              hostAvatar: hostUser ? hostUser.avatar : null
            };
          }
        } catch (error) {
          console.error(`Failed to fetch room details for room ${roomId}:`, error);
        }
        
        // 如果获取房间详情失败，返回基本房间信息
        return {
          roomId: roomId,
          host: 0, // 从详细API获取
          hostName: '未知用户', // 待实现：获取用户名
          players: [], // 从详细API获取
          isLocked: false, // 从详细API获取
        };
      });
      
      // 过滤掉无效的房间
      const allRoomDetails = await Promise.all(roomDetailsPromises);
      const newRooms = allRoomDetails.filter((room): room is any => room !== undefined && room !== null);
      
      // 断开已不存在房间的WebSocket连接
      const newRoomIds = newRooms.map(r => r.roomId);
      const oldRoomIds = rooms.value.map(r => r.roomId);
      
      for (const oldRoomId of oldRoomIds) {
        if (!newRoomIds.includes(oldRoomId)) {
          disconnectRoomWs(oldRoomId);
        }
      }
      
      // 更新房间列表
      rooms.value = newRooms;
      
      // 为新房间建立WebSocket连接
      for (const room of newRooms) {
        if (!roomWss[room.roomId]) {
          connectToRoomWs(room.roomId);
        }
      }
    }
  } catch (error) {
    console.error('Failed to refresh rooms list:', error);
  }
}

onMounted(() => {
  fetchServerDetail();
  // 连接WebSocket以接收实时更新
  connectToServerWs();
});

onUnmounted(() => {
  // 组件卸载时关闭WebSocket连接
  if (serverWs) {
    serverWs.close();
  }
  
  // 关闭所有房间WebSocket连接
  for (const roomId in roomWss) {
    roomWss[roomId].close();
  }
});

// 跳转到房间详情页
function goToRoomDetail(roomId: string) {
  router.push({ name: 'room-detail', params: { serverName: serverName, roomId: roomId } });
}


</script>

<template>
  <div class="flex flex-row justify-center w-full">
    <div class="mx-8 max-w-none w-full lg:w-2/3 flex flex-col">
      <!-- Loading state -->
      <div v-if="loading" class="flex justify-center items-center py-16">
        <span class="loading loading-spinner loading-lg"></span>
        <span class="ml-4 text-xl">{{ t('loading') }}</span>
      </div>

      <!-- Server details -->
      <div v-else-if="server" class="flex flex-col gap-4 mt-4">
        <!-- Server info card -->
        <div class="card bg-base-100 shadow-xl p-6">
          <h1 class="text-3xl font-bold mb-4">{{ server.name }}</h1>
          <div class="flex flex-col gap-2 text-base-content/70">
            <div class="flex items-center gap-2">
              <i class="fa-solid fa-network-wired"></i>
              <span>{{ server.address }}</span>
            </div>
            <div class="flex gap-4 mt-2">
              <div class="badge badge-primary badge-lg">
                <i class="fa-solid fa-door-open mr-2"></i>
                {{ server.roomCount }} 房间
              </div>
              <div class="badge badge-secondary badge-lg">
                <i class="fa-solid fa-users mr-2"></i>
                {{ server.currentUsers }} / {{ server.maxUsers }} 在线
              </div>
            </div>
          </div>
        </div>

        <!-- Online users -->
        <div class="card bg-base-100 shadow-xl p-6">
          <h2 class="text-2xl font-bold mb-4">{{ t('online-users') }}</h2>
          <div v-if="userIds.length === 0" class="text-center py-8 text-base-content/50">
            {{ t('no-users') }}
          </div>
          <div v-else class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
            <OnlineUserCard
              v-for="userId in userIds"
              :key="userId"
              :id="userId"
            />
          </div>
        </div>

        <!-- Rooms -->
        <div class="card bg-base-100 shadow-xl p-6">
          <h2 class="text-2xl font-bold mb-4">{{ t('rooms') }}</h2>
          <div v-if="server.roomCount === 0" class="text-center py-8 text-base-content/50">
            {{ t('no-rooms') }}
          </div>
          <div v-else>
            <div class="text-lg mb-2">活跃房间: {{ server.roomCount }}</div>
            <div class="grid grid-cols-1 gap-4 mt-4">
              <div 
                v-for="room in validRooms" 
                :key="room.roomId"
                :class="['card', 'bg-base-200', 'shadow-md', 'hover:shadow-lg', 'transition-shadow', 'p-3', room.isLocked ? 'bg-error/20' : 'bg-success/20']"
                @click="goToRoomDetail(room.roomId)"
                class="cursor-pointer"
              >
                <div class="flex flex-col gap-2">
                  <!-- 房间ID行 -->
                  <div class="flex items-center justify-between">
                    <h3 class="text-lg font-bold truncate">{{ room.roomId }}</h3>
                  </div>
                  
                  <!-- 房主信息和玩家数行 -->
                  <div class="flex items-center gap-2 text-sm">
                    <div class="avatar">
                      <div class="w-8 rounded-xl">
                        <img 
                          v-if="room.hostAvatar" 
                          :src="fileToURL(room.hostAvatar)" 
                          :alt="room.hostName || `用户${room.host}`" 
                          class="w-full h-full object-cover"
                        >
                        <i v-else class="fa-solid fa-user w-full h-full flex items-center justify-center bg-base-300 p-2"></i>
                      </div>
                    </div>
                    <div class="flex-1 min-w-0">
                      <div class="flex flex-col gap-1">
                        <div class="truncate">房主: {{ room.hostName || `用户${room.host}` }}</div>
                        <div class="text-base-content/70 flex items-center gap-1">
                          <i class="fa-solid fa-users text-xs"></i>
                          <span>{{ room.players ? room.players.length : 0 }} 玩家</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

