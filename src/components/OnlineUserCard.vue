<script setup lang="ts">
import { ref } from 'vue';
import { useFetchApi, userNameClass } from '../common';
import type { UserView } from '../model';
import UserAvatar from './UserAvatar.vue';

const props = defineProps<{ id: number }>();

const fetchApi = useFetchApi();

const user = ref<UserView>();
fetchApi(`/user/${props.id}`, {}, (u) => {
  user.value = u as UserView;
});
</script>

<template>
  <router-link
    :to="`/user/${props.id}`"
    class="card bg-base-200 p-4 hover:shadow-lg transition-shadow group">
    <div v-if="!user" class="flex justify-center items-center h-16">
      <span class="loading loading-spinner loading-sm"></span>
    </div>
    <div v-else class="flex items-center gap-3">
      <div class="avatar">
        <div class="w-12 mask mask-squircle">
          <UserAvatar :url="user.avatar" />
        </div>
      </div>
      <div class="flex-1 min-w-0">
        <div
          class="font-bold truncate group-hover:link"
          :class="[userNameClass(user.badges)]">
          <del v-if="user.login_banned">{{ user.name }}</del>
          <span v-else>{{ user.name }}</span>
        </div>
        <div v-if="user.bio" class="text-sm text-base-content/70 truncate">
          {{ user.bio }}
        </div>
        <div class="text-xs text-base-content/50">
          RKS: {{ user.rks.toFixed(2) }}
        </div>
      </div>
    </div>
  </router-link>
</template>

