<template>
  <div v-if="isListReady" class="container--assets">
    <div v-if="filteredTokens.length > 0" class="container">
      <div class="row">
        <div>
          <span class="text--title">{{ $t('assets.xcmAssets') }}</span>
        </div>
        <div />
      </div>

      <div v-for="t in filteredTokens" :key="t.symbol">
        <Erc20Currency v-if="!checkIsCbridgeToken(t) && t.isXC20" :token="t" :is-xcm="true" />
      </div>
    </div>

    <div class="container">
      <div class="row">
        <div>
          <span class="text--title">{{ $t('assets.assets') }}</span>
        </div>

        <div class="row--search-option">
          <div :class="isSearch && 'search--active'">
            <div class="box--search">
              <table class="table--search">
                <tr class="tr--search">
                  <td>
                    <input
                      v-model="search"
                      type="text"
                      placeholder="Search"
                      class="input--search"
                      @focus="setIsSearch(true)"
                      @blur="setIsSearch(false)"
                    />
                  </td>
                  <td>
                    <div class="icon--search">
                      <astar-icon-search />
                    </div>
                  </td>
                </tr>
              </table>
            </div>
          </div>
          <EvmAssetOptions
            :toggle-is-hide-small-balances="toggleIsHideSmallBalances"
            :is-hide-small-balances="isHideSmallBalances"
            :tokens="tokens"
          />
        </div>
      </div>

      <div class="border--separator" />

      <div v-if="isDisplayNativeToken" class="rows">
        <div class="row row--details">
          <div class="row__left">
            <div class="column--currency">
              <img class="token-logo" :src="nativeTokenImg" :alt="nativeTokenSymbol" />
              <div v-if="nativeTokenSymbol && currentNetworkName" class="column--ticker">
                <span class="text--title">{{ nativeTokenSymbol }}</span>
                <span class="text--label">{{ currentNetworkName }}</span>
              </div>
              <div v-else>
                <q-skeleton animation="fade" class="skeleton--md" />
              </div>
            </div>
          </div>
          <div class="row__right row__right--evm">
            <div class="column column--balance">
              <div class="column__box">
                <div class="text--accent">
                  <TokenBalance :balance="String(bal)" :symbol="nativeTokenSymbol" />
                </div>
                <div class="text--label">
                  <span>{{ $n(balUsd) }} {{ $t('usd') }}</span>
                </div>
              </div>
            </div>
            <div class="column--asset-buttons column--buttons--native-token">
              <router-link :to="buildTransferPageLink(nativeTokenSymbol)">
                <button class="btn btn--sm">
                  {{ $t('assets.transfer') }}
                </button>
              </router-link>
              <!-- Only SDN is able to bridge via cBridge at this moment -->
              <a
                v-if="nativeTokenSymbol === 'SDN'"
                :href="cbridgeAppLink"
                target="_blank"
                rel="noopener noreferrer"
              >
                <button class="btn btn--sm">
                  {{ $t('assets.bridge') }}
                </button>
              </a>

              <button
                v-if="isFaucet"
                class="btn btn--sm"
                @click="handleModalFaucet({ isOpen: true })"
              >
                {{ $t('assets.faucet') }}
              </button>
            </div>
          </div>
        </div>
      </div>

      <div v-for="t in filteredTokens" :key="t.symbol">
        <div v-if="checkIsCbridgeToken(t)">
          <EvmCbridgeToken v-if="t.symbol !== nativeTokenSymbol" :token="t" />
        </div>
        <div v-else>
          <Erc20Currency v-if="!t.isXC20" :token="t" />
        </div>
      </div>
      <div v-if="!filteredTokens && !isDisplayNativeToken" class="box--no-result">
        <span class="text--xl">{{ $t('assets.noResults') }}</span>
      </div>
    </div>
    <ModalFaucet :is-modal-faucet="isModalFaucet" :handle-modal-faucet="handleModalFaucet" />
  </div>
</template>
<script lang="ts">
import { ethers } from 'ethers';
import { $web3 } from 'src/boot/api';
import { cbridgeAppLink, checkIsCbridgeToken } from 'src/c-bridge';
import Erc20Currency from 'src/components/assets/Erc20Currency.vue';
import EvmAssetOptions from 'src/components/assets/EvmAssetOptions.vue';
import EvmCbridgeToken from 'src/components/assets/EvmCbridgeToken.vue';
import { getBalance } from 'src/config/web3';
import { useAccount, useBalance, useNetworkInfo, usePrice } from 'src/hooks';
import { Erc20Token, getTokenImage } from 'src/modules/token';
import { useStore } from 'src/store';
import { computed, defineComponent, PropType, ref, watchEffect } from 'vue';
import ModalFaucet from 'src/components/assets/modals/ModalFaucet.vue';
import { buildTransferPageLink } from 'src/router/routes';
import TokenBalance from 'src/components/common/TokenBalance.vue';

export default defineComponent({
  components: {
    EvmCbridgeToken,
    ModalFaucet,
    Erc20Currency,
    EvmAssetOptions,
    TokenBalance,
  },
  props: {
    tokens: {
      type: Object as PropType<Erc20Token[]>,
      required: false,
      default: null,
    },
  },
  setup(props) {
    const isModalFaucet = ref<boolean>(false);
    const isHideSmallBalances = ref<boolean>(false);
    const token = ref<Erc20Token | string | null>(null);
    const symbol = ref<string>('');
    const bal = ref<number>(0);
    const balUsd = ref<number>(0);
    const isShibuya = ref<boolean>(false);
    const isFaucet = ref<boolean>(false);
    const isSearch = ref<boolean>(false);
    const search = ref<string>('');
    // Memo: defined by hard-coding to avoid sending too many requests to faucet API server
    const mainnetFaucetAmount = 0.002;

    const { currentAccount } = useAccount();
    const { nativeTokenUsd } = usePrice();
    const { accountData } = useBalance(currentAccount);

    const store = useStore();
    const isH160 = computed<boolean>(() => store.getters['general/isH160Formatted']);
    const isLoading = computed<boolean>(() => store.getters['general/isLoading']);

    const { currentNetworkName, nativeTokenSymbol } = useNetworkInfo();

    const nativeTokenImg = computed<string>(() =>
      getTokenImage({ isNativeToken: true, symbol: nativeTokenSymbol.value })
    );
    const isListReady = computed<boolean>(() => !!(isShibuya.value || props.tokens.length > 0));

    const isDisplayNativeToken = computed<boolean>(() => {
      return (
        !search.value || nativeTokenSymbol.value.toLowerCase().includes(search.value.toLowerCase())
      );
    });

    const filteredTokens = computed<Erc20Token[] | null>(() => {
      if (!search.value) return props.tokens;
      if (!props.tokens) return null;
      const tokens = isHideSmallBalances.value
        ? props.tokens.filter((it) => Number(it.userBalance) > 0)
        : props.tokens;

      if (!search.value) return tokens;

      const value = search.value.toLowerCase();
      const result = tokens
        .map((token: Erc20Token) => {
          const isFoundToken =
            value === token.address.toLowerCase() ||
            token.symbol.toLowerCase().includes(value) ||
            token.name.toLowerCase().includes(value);
          return isFoundToken ? token : undefined;
        })
        .filter((it) => it !== undefined) as Erc20Token[];
      return result.length > 0 ? result : null;
    });

    const toggleIsHideSmallBalances = (): void => {
      isHideSmallBalances.value = !isHideSmallBalances.value;
    };

    const handleModalFaucet = ({ isOpen }: { isOpen: boolean }): void => {
      isModalFaucet.value = isOpen;
    };

    const setIsSearch = (isTyping: boolean): void => {
      isSearch.value = isTyping;
    };

    const updateStates = async (): Promise<void> => {
      if (isLoading.value || !nativeTokenSymbol.value || !isH160.value) return;
      try {
        const balWei = await getBalance($web3.value!, currentAccount.value);
        bal.value = Number(ethers.utils.formatEther(balWei));
        isShibuya.value = nativeTokenSymbol.value === 'SBY';
        isFaucet.value = isShibuya.value || mainnetFaucetAmount > bal.value;
        if (nativeTokenUsd.value) {
          balUsd.value = nativeTokenUsd.value * bal.value;
        }
      } catch (error: any) {
        console.error(error.message);
      }
    };

    watchEffect(async () => {
      await updateStates();
    });

    return {
      bal,
      nativeTokenSymbol,
      balUsd,
      currentNetworkName,
      mainnetFaucetAmount,
      isFaucet,
      symbol,
      token,
      nativeTokenImg,
      isListReady,
      isModalFaucet,
      isSearch,
      search,
      filteredTokens,
      isDisplayNativeToken,
      accountData,
      cbridgeAppLink,
      isHideSmallBalances,
      isLoading,
      buildTransferPageLink,
      setIsSearch,
      handleModalFaucet,
      checkIsCbridgeToken,
      toggleIsHideSmallBalances,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/assets/styles/asset-list.scss';
</style>
