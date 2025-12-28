<i18n>
en:
  title: Multiplayer Servers
  no-servers: No servers available
  loading: Loading servers...

zh-CN:
  title: 多人联机服务器
  no-servers: 暂无服务器
  loading: 加载服务器中...
</i18n>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useI18n } from 'vue-i18n';
import type { Server } from '@/model';
import { MPAPI_BASE } from '@/common';
import ServerCard from '../components/ServerCard.vue';

const { t } = useI18n();

const servers = ref<Server[]>([]);
const loading = ref(true);



// 获取服务器列表
async function fetchServers() {
  loading.value = true;

  try {
    const response = await fetch(`${MPAPI_BASE}/api/Server/status`);
    const data = await response.json();

    if (data.totalServers !== undefined && data.servers !== undefined) {
      // 先使用从 /api/Server/status 获取的基本信息创建服务器列表
      const basicServers = data.servers.map((apiServer: any) => {
        return {
          id: 0, // API 中未提供具体 ID，使用 0 作为占位符
          name: apiServer.serverName,
          address: `${apiServer.serverName}.phira-mp.com:12345`, // 构造地址
          roomCount: 0, // 房间数量需要从单独的 API 获取
          maxUsers: 0, // 最大用户数需要从 /api/Server/{serverName}/details 获取
          currentUsers: 0, // 当前用户数需要从 /api/Server/{serverName}/details 获取
          uptime: 0, // 运行时间需要从 /api/Server/{serverName}/details 获取
        };
      });
      
      // 为每个服务器获取详细信息
      servers.value = await Promise.all(basicServers.map(async (basicServer: Server) => {
        try {
          const detailsResponse = await fetch(`${MPAPI_BASE}/api/Server/${basicServer.name}/details`);
          const detailsData = await detailsResponse.json();
          
          if (detailsData.success) {
            // 获取房间信息
            const roomsResponse = await fetch(`${MPAPI_BASE}/api/Room/${basicServer.name}`);
            const roomsData = await roomsResponse.json();
            
            // 获取服务器中的玩家列表
            const playersResponse = await fetch(`${MPAPI_BASE}/api/Player/${basicServer.name}`);
            const playersData = await playersResponse.json();
            
            // 返回包含详细信息的服务器对象
            return {
              ...basicServer,
              address: detailsData.externalAddress || `${basicServer.name}.phira-mp.com:12345`, // 使用API返回的外部地址
              maxUsers: detailsData.maxPlayers,
              currentUsers: playersData.success ? playersData.players.length : detailsData.currentPlayers,
              uptime: detailsData.uptime, // API 现在直接返回秒数
              roomCount: roomsData.success ? roomsData.roomCount : 0,
            };
          }
        } catch (error) {
          console.error(`Failed to fetch details for server ${basicServer.name}:`, error);
        }
        
        // 如果获取详细信息失败，返回基本服务器信息
        return basicServer;
      }));
    }
  } catch (error) {
    console.error('Failed to fetch servers:', error);
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  fetchServers();
});

</script>

<template>
  <div class="flex flex-row justify-center w-full">
    <div class="mx-8 max-w-none w-full lg:w-1/2 flex flex-col items-end">
      <div class="card bg-base-100 shadow-xl p-4 mt-4 w-full">
        <h1 class="text-3xl font-bold mb-6">{{ t('title') }}</h1>

        <!-- Loading state -->
        <div v-if="loading" class="flex justify-center items-center py-16">
          <span class="loading loading-spinner loading-lg"></span>
          <span class="ml-4 text-xl">{{ t('loading') }}</span>
        </div>

        <!-- Empty state -->
        <div v-else-if="servers.length === 0" class="text-center py-16">
          <i class="fa-solid fa-server text-6xl text-base-300 mb-4"></i>
          <p class="text-xl text-base-content/70">{{ t('no-servers') }}</p>
        </div>

        <!-- Server list -->
        <div v-else class="flex flex-col gap-4">
          <ServerCard
            v-for="server in servers"
            :key="server.id"
            :server="server"
          />
        </div>
      </div>
    </div>
  </div>
</template>

