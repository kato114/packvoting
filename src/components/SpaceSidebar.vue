<script setup lang="ts">
import { ExtendedSpace } from '@/helpers/interfaces';
import SpaceSidebarSubspaces from './SpaceSidebarSubspaces.vue';

defineProps<{ space: ExtendedSpace }>();
</script>

<template>
  <div class="mb-4">
    <BaseBlock slim class="overflow-hidden">
      <div class="lg:max-h-[calc(100vh-120px)] lg:overflow-y-auto">
        <!-- <SpaceSidebarHeader :space="space" /> -->
        <SpaceSidebarNavigation :space="space" class="lg:mt-0" />
        <!-- <SpaceSidebarSubspaces :space="space" class="lg:flex" />
        <SpaceSidebarFooter :space="space" class="lg:flex" /> -->
      </div>
    </BaseBlock>
  </div>
</template>
