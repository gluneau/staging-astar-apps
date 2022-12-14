<template>
  <div v-if="!isLoading" class="wrapper--assets">
    <div class="container--assets">
      <Account
        :ttl-erc20-amount="evmAssets.ttlEvmUsdAmount"
        :ttl-native-xcm-usd-amount="ttlNativeXcmUsdAmount"
        :is-loading-erc20-amount="isLoading"
        :is-loading-xcm-assets-amount="isLoadingXcmAssetsAmount"
      />
      <div>
        <div v-if="isH160">
          <EvmAssetList :tokens="evmAssets.assets" />
        </div>
        <div v-else class="container--assets">
          <XcmNativeAssetList v-if="isEnableXcm" :xcm-assets="xcmAssets.assets" />
          <NativeAssetList />
        </div>
      </div>
    </div>
  </div>
</template>
<script lang="ts">
import Account from 'src/components/assets/Account.vue';
import EvmAssetList from 'src/components/assets/EvmAssetList.vue';
import NativeAssetList from 'src/components/assets/NativeAssetList.vue';
import XcmNativeAssetList from 'src/components/assets/XcmNativeAssetList.vue';
import { endpointKey, providerEndpoints } from 'src/config/chainEndpoints';
import { LOCAL_STORAGE } from 'src/config/localStorage';
import { useAccount, useBalance, useNetworkInfo } from 'src/hooks';
import { useStore } from 'src/store';
import { EvmAssets, XcmAssets } from 'src/store/assets/state';
import { Asset } from 'src/v2/models';
import { computed, defineComponent, ref, watch, watchEffect } from 'vue';

export default defineComponent({
  components: {
    Account,
    NativeAssetList,
    EvmAssetList,
    XcmNativeAssetList,
  },
  setup() {
    const token = ref<Asset | null>(null);
    const isModalXcmBridge = ref<boolean>(false);
    const isModalXcmTransfer = ref<boolean>(false);

    const store = useStore();
    const { currentAccount } = useAccount();
    const { accountData } = useBalance(currentAccount);
    const { isMainnet, currentNetworkIdx } = useNetworkInfo();
    const evmNetworkId = computed(() => {
      return Number(providerEndpoints[currentNetworkIdx.value].evmChainId);
    });
    const isLoading = computed<boolean>(() => store.getters['general/isLoading']);
    const isH160 = computed(() => store.getters['general/isH160Formatted']);
    const isShibuya = computed(() => currentNetworkIdx.value === endpointKey.SHIBUYA);

    const xcmAssets = computed<XcmAssets>(() => store.getters['assets/getAllAssets']);
    const ttlNativeXcmUsdAmount = computed<number>(() => xcmAssets.value.ttlNativeXcmUsdAmount);
    const evmAssets = computed<EvmAssets>(() => store.getters['assets/getEvmAllAssets']);

    const isLoadingXcmAssetsAmount = computed<boolean>(() => {
      if (isMainnet.value) {
        return !xcmAssets.value.assets.length;
      } else {
        return false;
      }
    });

    const handleUpdateXcmTokenAssets = () => {
      currentAccount.value &&
        store.dispatch('assets/getAssets', { address: currentAccount.value, isFetchUsd: true });
    };

    const handleUpdateEvmAssets = (): void => {
      if (isH160.value) {
        currentAccount.value &&
          store.dispatch('assets/getEvmAssets', {
            currentAccount: currentAccount.value,
            srcChainId: evmNetworkId.value,
            currentNetworkIdx: currentNetworkIdx.value,
            isFetchUsd: true,
          });
      }
    };

    // Memo: triggered after users have imported custom ERC20 tokens
    const handleImportingCustomToken = async (): Promise<void> => {
      if (!isH160.value) return;
      window.addEventListener(LOCAL_STORAGE.EVM_TOKEN_IMPORTS, () => {
        handleUpdateEvmAssets();
      });
    };

    watch([currentAccount], handleUpdateXcmTokenAssets, { immediate: true });
    watch([currentAccount], handleUpdateEvmAssets, { immediate: true });
    watchEffect(handleImportingCustomToken);

    const isEnableXcm = computed(
      () => !isShibuya.value && xcmAssets.value.assets && xcmAssets.value.assets.length > 0
    );

    const handleEvmAssetLoader = (): void => {
      if (isMainnet.value && isH160.value) {
        const isAssets = evmAssets.value.assets.length > 0;
        store.commit('general/setLoading', !isAssets);
      }
    };

    watch([evmAssets, xcmAssets, isH160, isMainnet], handleEvmAssetLoader, { immediate: true });

    return {
      evmAssets,
      isLoadingXcmAssetsAmount,
      isH160,
      isEnableXcm,
      xcmAssets,
      ttlNativeXcmUsdAmount,
      isModalXcmTransfer,
      token,
      accountData,
      isModalXcmBridge,
      isLoading,
      handleUpdateXcmTokenAssets,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/assets/styles/assets.scss';
</style>
