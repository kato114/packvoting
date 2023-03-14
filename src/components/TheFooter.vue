<script setup lang="ts">
const yearNow = new Date().getFullYear();
</script>

<template>
  <div class="-mt-3 border-t" />
  <BaseContainer class="space-y-5 py-[30px] md:space-y-0">
    <div class="flex flex-col items-center justify-between gap-4 md:flex-row">
      <div class="whitespace-nowrap text-center opacity-40 md:text-left">
        © {{ yearNow }} PackToken DAO.
      </div>
      <div class="">
        <div class="space-y-5 md:flex md:space-y-0">
          <div class="pb-1 md:hidden lg:mt-0 lg:block">
            <FooterSocials />
          </div>
        </div>
      </div>
    </div>
  </BaseContainer>
</template>
