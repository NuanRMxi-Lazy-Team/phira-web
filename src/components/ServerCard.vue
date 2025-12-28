<i18n>
en:
  rooms: Rooms
  online: Online
  uptime: Uptime
  days: days
  hours: hours
  minutes: minutes
  address: Copy Address
  copied: Copied to clipboard!
  copy-failed: Failed to copy

zh-CN:
  rooms: 房间数
  online: 在线人数
  uptime: 运行时长
  days: 天
  hours: 小时
  minutes: 分钟
  address: 复制连接地址
  copied: 已复制到剪贴板！
  copy-failed: 复制失败
</i18n>

<script setup lang="ts">
import { computed } from 'vue';
import { useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
import type { Server } from '@/model';
import { toast } from '@/common';

const props = defineProps<{ server: Server }>();

const { t } = useI18n();
const router = useRouter();

// 格式化运行时长
const formattedUptime = computed(() => {
  const minutes = Math.floor(props.server.uptime / 60);
  const hours = Math.floor(minutes / 60);
  const days = Math.floor(hours / 24);

  const remainingHours = hours % 24;
  const remainingMinutes = minutes % 60;

  const parts = [];
  if (days > 0) parts.push(`${days} ${t('days')}`);
  if (remainingHours > 0) parts.push(`${remainingHours} ${t('hours')}`);
  if (remainingMinutes > 0) parts.push(`${remainingMinutes} ${t('minutes')}`);

  return parts.join(' ');
});

// 计算在线人数百分比
const onlinePercentage = computed(() => {
  return (props.server.currentUsers / props.server.maxUsers) * 100;
});

// 根据在线人数百分比计算颜色类
const progressColorClass = computed(() => {
  const percentage = onlinePercentage.value;
  if (percentage < 30) return 'progress-success'; // 绿色：人少
  if (percentage < 60) return 'progress-info'; // 蓝色：适中
  if (percentage < 80) return 'progress-warning'; // 黄色：较多
  return 'progress-error'; // 红色：人多
});

// 复制地址到剪贴板
async function copyAddress(event: Event) {
  event.stopPropagation(); // 阻止事件冒泡，避免触发卡片点击
  try {
    await navigator.clipboard.writeText(props.server.address);
    toast(t('copied'), 'success');
  } catch (err) {
    toast(t('copy-failed'), 'error');
  }
}

// 跳转到服务器详情页面
function goToDetail() {
  router.push(`/mpserver/${props.server.name}`); // 使用服务器名称作为参数
}
</script>

<template>
  <div
    class="card bg-base-200 shadow-md hover:shadow-lg transition-shadow p-3 cursor-pointer hover:bg-base-300"
    @click="goToDetail">
    <div class="flex flex-col gap-2">
      <!-- 标题行 -->
      <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-1">
        <h2 class="text-lg font-bold truncate">{{ server.name }}</h2>
        <div class="flex items-center gap-1.5 text-xs text-base-content/70 flex-shrink-0">
          <i class="fa-solid fa-clock text-[10px]"></i>
          <span class="whitespace-nowrap">{{ formattedUptime }}</span>
        </div>
      </div>

      <!-- 地址行 -->
      <div class="flex items-center gap-2 text-xs">
        <i class="fa-solid fa-network-wired text-primary text-[10px]"></i>
        <span class="font-mono text-base-content/70 flex-1 truncate">{{ server.address }}</span>
        <button
          class="btn btn-xs btn-ghost btn-circle h-6 w-6 min-h-0"
          @click="copyAddress"
          :title="t('address')">
          <i class="fa-solid fa-copy text-[10px]"></i>
        </button>
      </div>

      <!-- 统计信息行 -->
      <div class="flex flex-col sm:flex-row gap-2 items-stretch">
        <!-- 房间数 -->
        <div class="bg-base-100 rounded p-2 flex items-center gap-2 flex-1 min-w-0">
          <div class="flex-shrink-0">
            <i class="fa-solid fa-door-open text-lg text-primary"></i>
          </div>
          <div class="min-w-0">
            <div class="text-[10px] text-base-content/70 leading-tight">{{ t('rooms') }}</div>
            <div class="text-lg font-bold leading-tight">{{ server.roomCount }}</div>
          </div>
        </div>

        <!-- 在线人数进度条 -->
        <div class="bg-base-100 rounded p-2 flex-[2] min-w-0">
          <div class="flex items-center justify-between mb-0.5">
            <span class="text-[10px] text-base-content/70">{{ t('online') }}</span>
            <span class="text-xs font-semibold">
              {{ server.currentUsers }} / {{ server.maxUsers }}
            </span>
          </div>
          <progress
            class="progress w-full h-2"
            :class="progressColorClass"
            :value="server.currentUsers"
            :max="server.maxUsers">
          </progress>
        </div>
      </div>
    </div>
  </div>
</template>

