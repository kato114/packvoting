<script setup lang="ts">
import { watch, ref, computed } from 'vue';
import { useRouter } from 'vue-router';
import draggable from 'vuedraggable';

import { lsSet, lsGet } from '@/helpers/utils';

import {
  useUnseenProposals,
  useExtendedSpaces,
  useFollowSpace,
  useSpaces,
  useWeb3,
  useApp
} from '@/composables';

const router = useRouter();

const { web3Account } = useWeb3();
const { loadFollows, followingSpaces, loadingFollows } = useFollowSpace();
const { spaceHasUnseenProposals } = useUnseenProposals();
const { domain, showSidebar } = useApp();
const { loadExtendedSpaces, extendedSpaces, spaceLoading } =
  useExtendedSpaces();
const { spaces } = useSpaces();

const draggableSpaces = ref<string[]>([]);

const extendedSpacesObj = computed(() => {
  return (
    extendedSpaces.value?.reduce(
      (acc, space) => ({ ...acc, [space.id]: space }),
      {}
    ) ?? {}
  );
});

function saveSpaceOrder() {
  if (web3Account.value)
    lsSet(
      `sidebarSpaceOrder.${web3Account.value.slice(0, 8).toLowerCase()}`,
      draggableSpaces.value
    );
}

watch(followingSpaces, () => {
  draggableSpaces.value = followingSpaces.value;
  const sidebarSpaceOrder = lsGet(
    `sidebarSpaceOrder.${web3Account.value.slice(0, 8).toLowerCase()}`,
    []
  );
  // Order side bar and add new spaces to the end of the sidebar
  draggableSpaces.value.sort((a, b) => {
    if (!sidebarSpaceOrder.includes(a)) return -1;
    if (!sidebarSpaceOrder.includes(b)) return 1;
    return sidebarSpaceOrder.indexOf(a) - sidebarSpaceOrder.indexOf(b);
  });

  saveSpaceOrder();
});

watch(followingSpaces, () => {
  loadExtendedSpaces(followingSpaces.value);
});

watch(
  web3Account,
  () => {
    loadFollows();
  },
  { immediate: true }
);
</script>

<template>
  <div
    class="no-scrollbar mt-3 flex h-full flex-col items-end overflow-auto overscroll-contain py-2"
    @click="showSidebar = false"
  >
    <div v-if="!domain" class="relative flex items-center px-2">
      <router-link :to="{ name: 'home' }">
        <ButtonSidebar class="!border-0">
          <img src="/coin_logo.png" />
        </ButtonSidebar>
      </router-link>
    </div>
    <div class="mt-2 px-2">
      <router-link :to="{ path: '/' }">
        <ButtonSidebar>
          <ISLenster />
        </ButtonSidebar>
      </router-link>
    </div>

    <div class="mt-2 flex flex-col items-center space-y-2 px-2">
      <router-link :to="{ path: '/pack-leader.eth' }">
        <ButtonSidebar>
          <ISVoted />
        </ButtonSidebar>
      </router-link>
    </div>
  </div>
</template>
