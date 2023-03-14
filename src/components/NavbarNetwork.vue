<script setup lang="ts">
import { ref, watch, watchEffect } from 'vue';
import { getInstance } from '@snapshot-labs/lock/plugins/vue3';
import { useWeb3 } from '@/composables';
const { handleChainChanged } = useWeb3();

const auth = getInstance();

let selectedNetwork = ref('Ethereum');
let loadingStatus = ref(false);

const networkList = {
  Ethereum: {
    chainId: '0x5',
    name: 'Ethereum',
    nativeCurrency: {
      name: 'Ethereum',
      symbol: 'ETH',
      decimals: 18
    },
    blockExplorerUrls: ['https://testnet.bscscan.com'],
    rpcUrls: ['https://bsc-dataseed1.binance.org/']
  },
  Shibarium: {
    chainId: '0x395',
    chainName: 'Shibarium',
    nativeCurrency: {
      name: 'Shibarium',
      symbol: 'BONE',
      decimals: 18
    },
    blockExplorerUrls: ['https://testnet.bscscan.com'],
    rpcUrls: ['https://puppynet.shibrpc.com']
  }
};

const handleAction = async e => {
  if (auth.web3 != undefined) {
    if (window.ethereum) {
      try {
        // check if the chain to connect to is installed
        await window.ethereum.request({
          method: 'wallet_switchEthereumChain',
          params: [{ chainId: networkList[e].chainId }] // chainId must be in hexadecimal numbers
        });
      } catch (error) {
        // This error code indicates that the chain has not been added to MetaMask
        // if it is not, then install it into the user MetaMask
        if (error.code === 4902) {
          try {
            await window.ethereum.request({
              method: 'wallet_addEthereumChain',
              params: [networkList[e]]
            });
          } catch (addError) {
            console.error(addError);
          }
        }
        console.error(error);
      }

      selectedNetwork.value = e;

      if (networkList[e] == undefined) {
        selectedNetwork.value = 'Wrong';
        return;
      }

      handleChainChanged(networkList[e].chainId);
    } else {
      // if no window.ethereum then MetaMask is not installed
      alert(
        'MetaMask is not installed. Please consider installing it: https://metamask.io/download.html'
      );
    }
  } else {
    selectedNetwork.value = e;
  }
};

watchEffect(async () => {
  if (auth.web3 != undefined) {
    if (auth.web3._network != undefined && auth.web3._network != null) {
      if (auth.web3._network.chainId == 917)
        selectedNetwork.value = 'Shibarium';
      else if (auth.web3._network.chainId == 5)
        selectedNetwork.value = 'Ethereum';
      else selectedNetwork.value = 'Wrong';
    }
  }
});

setInterval(() => {
  if (loadingStatus.value) loadingStatus.value = false;
  else loadingStatus.value = true;
}, 1000);

watch(
  loadingStatus,
  async () => {
    if (window.ethereum) {
      const chainId = await window.ethereum.request({ method: 'eth_chainId' });
      if (chainId == '0x5') selectedNetwork.value = 'Ethereum';
      else if (chainId == '0x395') selectedNetwork.value = 'Shibarium';
      else selectedNetwork.value = 'Wrong';
    }
  },
  { immediate: true }
);
</script>

<template>
  <BaseMenu
    :items="[
      {
        text: 'Ethereum',
        action: 'Ethereum',
        extras: { icon: '1.png' }
      },
      {
        text: 'Shibarium',
        action: 'Shibarium',
        extras: { icon: '917.png' }
      }
    ]"
    @select="handleAction($event)"
  >
    <template #button>
      <BaseButton class="mt-2 mb-1 flex items-center justify-center">
        <img src="/1.png" width="30" v-if="selectedNetwork === 'Ethereum'" />
        <img src="/917.png" width="30" v-if="selectedNetwork === 'Shibarium'" />
        <img src="/wrong.png" width="30" v-if="selectedNetwork === 'Wrong'" />
        <i-ho-chevron-down
          class="-mr-1 ml-1 text-xs text-skin-text"
          aria-hidden="true"
        />
      </BaseButton>
    </template>
    <template #item="{ item }">
      <div class="flex items-center space-x-2">
        <div class="w-[24px]">
          <img :src="item.extras.icon" width="30" />
        </div>
        <div>
          {{ item.text }}
        </div>
      </div>
    </template>
  </BaseMenu>
</template>
