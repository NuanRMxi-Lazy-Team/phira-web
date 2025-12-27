<script lang="ts">
import { ref, watch } from 'vue';

const onLoaded = ref<() => void>();
const component = ref();

watch(component, (comp) => {
  if (comp && onLoaded.value) onLoaded.value();
});

export function useOnLoaded() {
  return onLoaded;
}

export default {};
</script>

<script setup lang="ts">
import AppFooter from './components/AppFooter.vue';
import AppHeader from './components/AppHeader.vue';
import LoadView from './components/LoadView.vue';
import { useI18n } from 'vue-i18n';

const { t } = useI18n();
const showDisclaimer = ref<boolean>(localStorage.getItem('hideDisclaimer') !== 'true');

const closeDisclaimer = (): void => {
  showDisclaimer.value = false;
  localStorage.setItem('hideDisclaimer', 'true');
};
</script>

<template>
  <AppHeader />
  <div class="w-full mt-20">
    <router-view v-slot="{ Component }">
      <Suspense timeout="0">
        <template #default>
          <component :is="Component" ref="component" />
        </template>
        <template #fallback>
          <div class="flex justify-center">
            <LoadView />
          </div>
        </template>
      </Suspense>
    </router-view>
  </div>
  <AppFooter />

  <!-- Disclaimer Banner -->
  <transition name="slide-up">
    <div v-if="showDisclaimer" class="fixed bottom-0 left-0 right-0 z-50 alert alert-warning shadow-lg rounded-none flex justify-between items-center py-3 px-4">
      <div class="flex items-center gap-3">
        <svg xmlns="http://www.w3.org/2000/svg" class="stroke-current flex-shrink-0 h-6 w-6" fill="none" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
        </svg>
        <span>{{ t('disclaimer') }}</span>
      </div>
      <button @click="closeDisclaimer" class="btn btn-sm btn-ghost">
        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
        </svg>
      </button>
    </div>
  </transition>
</template>

<style scoped>
.slide-up-enter-active,
.slide-up-leave-active {
  transition: transform 0.3s ease-out;
}

.slide-up-enter-from {
  transform: translateY(100%);
}

.slide-up-leave-to {
  transform: translateY(100%);
}
</style>

