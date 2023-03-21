<script setup>
import { ref, watch, watchEffect } from 'vue';
import { _ } from 'lodash';
import Web3 from 'web3';

import { useMeta } from '@/composables';
import { getInstance } from '@snapshot-labs/lock/plugins/vue3';
import { sendTransaction, sleep } from '@snapshot-labs/snapshot.js/src/utils';
import { useWeb3, useFlashNotification } from '@/composables';

import ERC20 from '../abis/ERC20.json';
import DividendTracker from '../abis/DividendTracker.json';
import Aggregator from '../abis/Aggregator.json';

import { packToken, rewardTokenList } from '../constants/config';

const { web3Account } = useWeb3();

const auth = getInstance();
const { notify } = useFlashNotification();

const rpcUrl = 'https://mainnet.infura.io/v3/c9e3e4efe8e14113a8c60cc110d13f7a';
const web3 = new Web3(rpcUrl);

const packAddress = '0xb186035490C8602EaD853EC98bE05E3461521Ab2';
const dividendAddress = '0x91888D363ccf32E0F10A6AC9A96d94eC58dB06F3';
const ethAddress = '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2';
const deadAddress = '0x000000000000000000000000000000000000dEaD';
const lpAddress = '0xb6B2B09CeD473A11c8BE983Ce7838fd4D83a21E6';
const ethPriceAggregator = '0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419';

const packDecimals = 18;
const ethDecimals = 18;
const aggregatorDecimals = 8;

const ethContract = new web3.eth.Contract(ERC20, ethAddress);
const packContract = new web3.eth.Contract(ERC20, packAddress);
const dividendContract = new web3.eth.Contract(
  DividendTracker,
  dividendAddress
);
const ethPriceAggregatorContract = new web3.eth.Contract(
  Aggregator,
  ethPriceAggregator
);

let alertStatus = ref(true);
let loadingStatus = ref(false);
let chainIDStatus = ref('0x1');

let marketCap = ref(-1);
let liquidyValue = ref(-1);
let totalRewards = ref(-1);
let tokenPrice = ref(-1);

let userPackBalancer = ref(-1);
let userEarned = ref(-1);
let userPendingReward = ref(-1);

const getTokenInfo = async () => {
  if (chainIDStatus.value == '0x1') {
    let aaa = await ethPriceAggregatorContract.methods.latestAnswer().call();
    const ethPrice =
      (await ethPriceAggregatorContract.methods.latestAnswer().call()) /
      10 ** aggregatorDecimals;

    const ethBalanceInLP =
      (await ethContract.methods.balanceOf(lpAddress).call()) /
      10 ** ethDecimals;
    const packBalanceInLP =
      (await packContract.methods.balanceOf(lpAddress).call()) /
      10 ** packDecimals;

    const packPrice = (ethPrice * ethBalanceInLP) / packBalanceInLP;

    const packTotalSupply =
      (await packContract.methods.totalSupply().call()) / 10 ** packDecimals;
    const packDeadSupply =
      (await packContract.methods.balanceOf(deadAddress).call()) /
      10 ** packDecimals;
    const packCircSupply = packTotalSupply - packDeadSupply;

    marketCap.value = packCircSupply * packPrice;
    liquidyValue.value = ethBalanceInLP * ethPrice;
    tokenPrice.value = packPrice;

    totalRewards.value =
      ((await dividendContract.methods.totalDividendsDistributed().call()) *
        ethPrice) /
      10 ** ethDecimals;
  } else {
    marketCap.value = 1000;
    liquidyValue.value = 1000;
    tokenPrice.value = 0.000001;
    totalRewards.value = 1000;
  }
};

const getUserInfo = async account => {
  if (account !== undefined && account !== null && account.length > 0) {
    if (chainIDStatus.value == '0x1') {
      const ethPrice =
        (await ethPriceAggregatorContract.methods.latestAnswer().call()) /
        10 ** aggregatorDecimals;

      userPackBalancer.value =
        (await packContract.methods.balanceOf(account).call()) /
        10 ** packDecimals;

      const { totalDividends, withdrawableDividends } =
        await dividendContract.methods.getAccount(account).call();
      userEarned.value = (totalDividends * ethPrice) / 10 ** ethDecimals;
      userPendingReward.value =
        (withdrawableDividends * ethPrice) / 10 ** ethDecimals;
    } else {
      userPackBalancer.value = 0;
      userEarned.value = 0;
      userPendingReward.value = 0;
    }
  }
};

const initUserInfo = () => {
  userPackBalancer.value = -1;
  userEarned.value = -1;
  userPendingReward.value = -1;
};

const initTokenInfo = () => {
  marketCap.value = -1;
  liquidyValue.value = -1;
  tokenPrice.value = -1;
  totalRewards.value = -1;
};

const handleAddCoin = async token => {
  const account = web3Account.value;
  if (account !== undefined && account !== null && account.length > 0) {
    try {
      const wasAdded = await ethereum.request({
        method: 'wallet_watchAsset',
        params: {
          type: 'ERC20',
          options: {
            address: token.address,
            symbol: token.name,
            decimals: token.decimals,
            image: token.image
          }
        }
      });

      if (wasAdded) {
        notify('Token added to wallet successfully.');
      }
    } catch (e) {
      console.log(e);
    }
  }
};

const handleCloseAlert = async () => {
  alertStatus.value = false;
  loadingStatus.value = true;
};

const handleClaim = async () => {
  const account = web3Account.value;
  if (account !== undefined && account !== null && account.length > 0) {
    try {
      const gasPrice = await web3.eth.getGasPrice();
      const block = await web3.eth.getBlock("latest");
      const gasLimit = (block.transactions.length == 0) ? block.gasLimit : Math.floor(2*(block.gasLimit/block.transactions.length));

      const tx = await sendTransaction(
        auth.web3,
        dividendAddress,
        DividendTracker,
        'claimReward',
        [],
        {
          gasPrice: gasPrice,
          gasLimit: gasLimit
        }
      );

      const receipt = await tx.wait();
      await sleep(3e3);
      notify('Claim reward succeed.');
    } catch (e) {
      notify(['red', 'Claim reward failed.']);
      console.log(e);
    }
  }
};

watch(
  web3Account,
  async () => {
    if (chainIDStatus.value != -1) {
      if (chainIDStatus.value == '0x1' || chainIDStatus.value == '0x395') {
        getUserInfo(web3Account.value);
      } else {
        initUserInfo();
      }
    }
  },
  { immediate: true }
);

watch(
  chainIDStatus,
  () => {
    if (chainIDStatus.value != -1) {
      if (chainIDStatus.value == '0x1' || chainIDStatus.value == '0x395') {
        getTokenInfo(chainIDStatus);
        getUserInfo(web3Account.value);
      } else {
        initTokenInfo();
        initUserInfo();
      }
    }
  },
  { immediate: true }
);

setInterval(() => {
  if (loadingStatus.value) loadingStatus.value = false;
  else loadingStatus.value = true;
}, 1000);

watch(
  loadingStatus,
  async () => {
    if (window.ethereum) {
      chainIDStatus.value = await window.ethereum.request({
        method: 'eth_chainId'
      });
    }
  },
  { immediate: true }
);

const formatPrice = (num, decimals) => {
  //return parseFloat(num.toFixed(decimals)).toLocaleString();
  return new Intl.NumberFormat('ja-JP').format(
    parseFloat(num).toFixed(decimals)
  );
};
</script>

<template>
  <BaseContainer class="gap-5 px-4">
    <div
      v-if="alertStatus"
      class="shadow-md relative rounded-b border-t-4 border-amber-500 bg-amber-100 px-4 py-3 text-amber-900"
      role="alert"
    >
      <div>
        <p class="font-bold">THE PACK GROWS STRONGER</p>
        <p class="text-sm">Get your Voting Power on PackToken DAO !</p>
      </div>
      <span
        class="absolute top-0 bottom-0 right-0 px-4 py-3"
        @click="handleCloseAlert()"
      >
        <svg
          class="text-red-500 h-3 w-3 fill-current"
          role="button"
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 20 20"
        >
          <title>Close</title>
          <path
            d="M14.348 14.849a1.2 1.2 0 0 1-1.697 0L10 11.819l-2.651 3.029a1.2 1.2 0 1 1-1.697-1.697l2.758-3.15-2.759-3.152a1.2 1.2 0 1 1 1.697-1.697L10 8.183l2.651-3.031a1.2 1.2 0 1 1 1.697 1.697l-2.758 3.152 2.758 3.15a1.2 1.2 0 0 1 0 1.698z"
          />
        </svg>
      </span>
    </div>
    <div class="mt-3 border-t" />
    <div
      class="mt-4 grid grid-cols-1 grid-rows-4 gap-4 rounded-lg border-2 border-solid border-amber-500 bg-zinc-900 md:grid-cols-4 md:grid-rows-1"
    >
      <div class="p-4 text-center md:text-center">
        <h6 class="text-amber-500">MarketCap</h6>
        <h1>${{ marketCap == -1 ? '---' : formatPrice(marketCap, 2) }}</h1>
      </div>
      <div class="p-4 text-center md:text-center">
        <h6 class="text-amber-500">Liquidity Value</h6>
        <h1>
          ${{ liquidyValue == -1 ? '---' : formatPrice(liquidyValue, 2) }}
        </h1>
      </div>
      <div class="p-4 text-center md:text-center">
        <h6 class="text-amber-500">Total Rewards</h6>
        <h1>
          ${{ totalRewards == -1 ? '---' : formatPrice(totalRewards, 2) }}
        </h1>
      </div>
      <div class="p-4 text-center md:text-center">
        <h6 class="text-amber-500">Pack Price</h6>
        <h1>${{ tokenPrice == -1 ? '---' : tokenPrice.toFixed(7) }}</h1>
      </div>
    </div>
    <div class="mt-3 border-t" />
    <div
      class="mt-4 grid grid-cols-1 grid-rows-2 gap-4 md:grid-cols-2 md:grid-rows-1"
    >
      <div
        class="flex flex-col items-center justify-between rounded-lg border-2 border-solid border-amber-500 bg-zinc-900 px-4 py-2 text-center md:flex-row md:items-start md:text-left"
      >
        <div>
          <h3 class="text-amber-500">Your $PACK Balance</h3>
          <h1>
            {{
              userPackBalancer == '-1'
                ? '---'
                : formatPrice(userPackBalancer, 2)
            }}
          </h1>
        </div>
        <div
          class="flex items-center justify-center gap-4 py-3 text-center md:flex-col md:gap-2 md:py-0"
        >
          <img :src="packToken.image" class="mx-auto mb-2" width="60" />
          <a
            class="rounded-full bg-amber-500 py-1 px-4 text-white hover:bg-amber-600 focus:outline-none"
            href="https://app.uniswap.org/#/swap?inputCurrency=ETH&outputCurrency=0xb186035490C8602EaD853EC98bE05E3461521Ab2"
            target="_blank"
          >
            Buy {{ packToken.name }}
          </a>
        </div>
      </div>
      <div
        class="flex flex-col items-center justify-center rounded-lg border-2 border-solid border-amber-500 bg-zinc-900 px-4 py-2 md:flex-col md:items-start md:justify-start"
      >
        <h3 class="text-amber-500">You Earned</h3>
        <h1>${{ userEarned == '-1' ? '---' : formatPrice(userEarned, 2) }}</h1>
      </div>
    </div>
    <div>
      <div class="mt-4 flex flex-col md:flex-row">
        <div
          class="flex basis-3/4 flex-col items-center justify-around border-2 border-solid border-amber-500 bg-zinc-900 md:flex-row"
        >
          <div class="py-5 text-center" v-for="rewardToken in rewardTokenList">
            <img :src="rewardToken.image" class="mx-auto mb-2" width="60" />
            <h4 class="mb-3 text-amber-500">{{ rewardToken.name }}</h4>
            <button
              class="rounded-full bg-amber-500 py-1 px-4 text-white hover:bg-amber-600 focus:outline-none disabled:bg-zinc-400"
              @click="handleAddCoin(rewardToken)"
              :disabled="rewardToken.name == 'Coming Soon'"
            >
              Add
              {{
                rewardToken.name == 'Coming Soon' ? 'Token' : rewardToken.name
              }}
            </button>
          </div>
        </div>
        <div
          class="order-first flex basis-1/4 flex-col items-center justify-center border-2 border-solid border-amber-500 bg-zinc-900 py-5 md:order-last"
        >
          <h3 class="text-amber-500">Pending Rewards</h3>
          <h1 class="mb-2">
            ${{
              userPendingReward == '-1' ? '---' : userPendingReward.toFixed(2)
            }}
          </h1>
          <button
            class="rounded-full bg-amber-500 py-1 px-4 text-white hover:bg-amber-600 disabled:bg-amber-500"
            :disabled="userPendingReward <= 0"
            @click="handleClaim()"
          >
            Claim
          </button>
        </div>
      </div>
    </div>
  </BaseContainer>
</template>
