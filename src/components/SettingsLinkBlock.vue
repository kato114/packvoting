<script setup lang="ts">
import schemas from '@snapshot-labs/snapshot.js/src/schemas';
import { useFormSpaceSettings } from '@/composables';

const props = defineProps<{
  context: 'setup' | 'settings';
}>();

const { form, getValidation } = useFormSpaceSettings(props.context);
</script>

<template>
  <BaseBlock :title="$t('settings.links')">
    <div class="space-y-3">
      <ContainerParallelInput>
        <InputSocial
          v-model="form.twitter"
          title="Twitter"
          :error="getValidation('twitter')"
          :max-length="schemas.space.properties.twitter.maxLength"
          icon="twitter"
          placeholder="e.g. elonmusk"
        />
        <InputSocial
          v-model="form.github"
          title="Github"
          :error="getValidation('github')"
          :max-length="schemas.space.properties.github.maxLength"
          icon="github"
          placeholder="e.g. vbuterin"
        />
        <InputSocial
          v-model="form.coingecko"
          title="CoinGecko"
          :error="getValidation('coingecko')"
          :max-length="schemas.space.properties.coingecko.maxLength"
          icon="coingecko"
          placeholder="e.g. uniswap"
        />
      </ContainerParallelInput>
    </div>
  </BaseBlock>
</template>
