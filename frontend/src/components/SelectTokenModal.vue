<template>
  <Modal v-model="dialogVisibleSelectToken" :customClass="customClass" :width="width">
    <template v-slot:header>
      {{ $t('swap.selectToken') }}
      <span style="margin-left: 4px">
        <div style="display: inline-block">
          <HelpTooltip
            placement="bottom"
            :content="$t('swap.helpTip')"
          ></HelpTooltip>
        </div>
      </span>
    </template>
    <div class="select-token-model-body">
      <div class="select-token-model-search">
        <input
          type="text"
          id="token-search-input"
          :placeholder="$t('swap.searchTip')"
          class="select-token-model-input"
          v-model="tokenNameOrAddress"
        />
      </div>
      <div class="select-token-model-filter">
        <div class="select-token-model-filter-text">{{ $t('swap.tokenName') }}</div>
        <div class="select-token-model-filter-button">
          <div class="select-token-model-filter-text">↓</div>
        </div>
      </div>
      <div class="select-token-model-list">
        <template v-if="searchTokenList.length <= 0 && tokenNameOrAddress === ''">
          <SelectTokenListItem
            v-for="(item, idx) in processedOriginalList"
            :key="idx"
            :symbol="item.dsymbol"
            :name="item.symbol"
            :logo="item.logo"
            :tokenAddress="item.address"
            :isYToken="item.tag === 'y3dToken'"
            @on-click="selectToken(item)"
          />
        </template>
        <template v-else>
          <SelectTokenListItem
            v-for="(item, idx) in searchTokenList"
            :key="idx"
            :symbol="item.dsymbol"
            :name="item.name"
            :logo="item.logo"
            :tokenAddress="item.address"
            :isYToken="item.tag === 'y3dToken'"
            @on-click="selectToken(item)"
          />
        </template>
      </div>
    </div>
  </Modal>
</template>

<script lang="ts">
/* eslint-disable no-unused-vars */
import Vue from 'vue';
import {
  debounce, sortBy, find, filter, DebouncedFunc,
} from 'lodash';
import { mapState } from 'vuex';
import { Contract } from 'ethers';
import { IERC20DetailInfo, fetchERC20Detail as fetchERC20DetailInfo } from '../utils/contract/fetchContractInfo';
import Modal from './Modal.vue';
import HelpTooltip from './HelpTooltip.vue';
import SelectTokenListItem from './SelectTokenListItem.vue';
import { CommonERC20 } from '../contract';
import { getProvider, utils } from '../store/ethers/ethersConnect';

export interface ITokenListItem {
  symbol: string
  dsymbol: string
  address: string
  tag: string
  chainId: number
}

export type ISearchTokenListItem = ITokenListItem & IERC20DetailInfo

export interface PairListModel {
  name: string
  uToken: string
  y3dToken: string
  yaddress: string
  uaddress: string
}

export interface ISelectTokenModalProps {
  isUToken: boolean
  value: boolean
  customClass: string
  width: string
  symbol: string
  commonBasesShow: boolean
  otherTokenInfo: ITokenListItem
  tokenInfo: ITokenListItem
  pairList: Array<PairListModel>
}

export interface ISelectTokenModalData {
  dialogVisibleSelectToken: boolean
  tokenNameOrAddress: string
  searchTokenList: Array<ISearchTokenListItem>
  commonBases: Array<any>
  originalList: Array<ITokenListItem>
}

export interface ISelectTokenModalMethods {
  reloadOriginalList: () => void
  searchToken: DebouncedFunc<(val: string) => Promise<void>>
  selectToken: (item: ITokenListItem) => void
  fetchBalanceOfDefaultList: () => Promise<void>
  getERC20: (_address: string) => Contract
  fetchERC20Detail: (item: ITokenListItem) => Promise<ISearchTokenListItem>
}

export interface ISelectTokenModalComputed {
  tokensInfo: Array<ITokenListItem>
  uTokenList: Array<ITokenListItem>
  y3dTokenList: Array<ITokenListItem>
  address: string
  network: string
  networkChainID: number
  processedOriginalList: Array<ITokenListItem>
}

export default Vue.extend<
  ISelectTokenModalData,
  ISelectTokenModalMethods,
  ISelectTokenModalComputed,
  ISelectTokenModalProps
>({
  components: {
    Modal,
    HelpTooltip,
    SelectTokenListItem,
  },
  props: {
    otherTokenInfo: {
      type: Object,
      default: null,
    },
    tokenInfo: {
      type: Object,
      default: null,
    },
    pairList: {
      type: Array,
      default: () => [],
    },
    isUToken: {
      type: Boolean,
      default: false,
    },
    value: {
      type: Boolean,
      default: false,
    },
    customClass: {
      type: String,
      default: '',
    },
    width: {
      type: String,
      default: '480px',
    },
    // 根据symbol知道是那个地方触发的modal
    symbol: {
      type: String,
      default: '',
    },
    commonBasesShow: {
      type: Boolean,
      default: false,
    },
  },
  mounted() {
    if (this.originalList.length === 0) {
      this.reloadOriginalList();
    }
  },
  data(): ISelectTokenModalData {
    return {
      dialogVisibleSelectToken: this.value,
      tokenNameOrAddress: '',
      searchTokenList: [],
      commonBases: [],
      originalList: [],
    };
  },
  computed: {
    ...mapState('ethers', ['address', 'network']),
    ...mapState('swap', ['tokensInfo', 'uTokenList', 'y3dTokenList']),
    networkChainID() {
      if (this.network === 'Mainnet') {
        return 1;
      }
      if (this.network === 'BSC Main Network') {
        return 56;
      }
      return 4;
    },
    processedOriginalList() {
      const filterTagToken = filter(this.originalList, (v: ITokenListItem) => {
        if (this.isUToken) {
          return v.tag === 'uToken';
        }
        return v.tag !== 'uToken';
      });
      if (this.isUToken) {
        return filterTagToken;
      }
      if (this.otherTokenInfo) {
        const usablePairList = filter(
          this.pairList,
          (v: PairListModel) => v.uToken === this.otherTokenInfo.dsymbol
            && v.uaddress === this.otherTokenInfo.address,
        ).flatMap((v: PairListModel) => v.y3dToken);
        const filterPairToken = filter(
          filterTagToken,
          (v: ITokenListItem) => usablePairList.indexOf(v.dsymbol) !== -1,
        );
        return filterPairToken;
      }
      return filterTagToken;
    },
  },
  watch: {
    tokensInfo() {
      this.reloadOriginalList();
    },
    uTokenList() {
      this.reloadOriginalList();
    },
    y3dTokenList() {
      this.reloadOriginalList();
    },
    otherTokenInfo() {
      if (!this.isUToken) {
        if (this.tokenInfo) {
          const fined = find(
            this.processedOriginalList,
            (v) => v.address === this.tokenInfo.address,
          );
          if (fined) {
            return;
          }
        }
        if (this.processedOriginalList && this.processedOriginalList.length > 0) {
          this.$emit('select-token', {
            data: this.processedOriginalList[0],
            symbol: this.symbol,
          });
        } else {
          this.$emit('clear-select-token');
        }
        this.dialogVisibleSelectToken = false;
      }
    },
    network() {
    },
    value(newVal) {
      this.dialogVisibleSelectToken = newVal;
      if (newVal) this.fetchBalanceOfDefaultList();
    },
    dialogVisibleSelectToken(newVal) {
      this.$emit('input', newVal);
      if (!newVal) {
        this.searchTokenList = [];
        this.tokenNameOrAddress = '';
      }
      setTimeout(() => {
        this.fetchBalanceOfDefaultList();
      }, 1000);
    },
    tokenNameOrAddress(newVal) {
      if (newVal.trim()) {
        this.searchToken(newVal);
      } else {
        this.searchTokenList = [];
      }
    },
  },
  methods: {
    reloadOriginalList() {
      const filterTokensInfo = filter(this.tokensInfo, (v) => v.chainId === this.networkChainID);
      this.originalList = this.uTokenList.concat(this.y3dTokenList).concat(filterTokensInfo);
      this.fetchBalanceOfDefaultList();
    },
    // eslint-disable-next-line func-names
    searchToken: debounce(async function (val: string) {
      const self = this as ISelectTokenModalData &
      ISelectTokenModalMethods &
      ISelectTokenModalComputed;
      if (val.indexOf('0x') === -1) {
        const filterResult = filter(
          self.processedOriginalList,
          (o: ITokenListItem) => o.symbol.indexOf(val) !== -1,
        );
        if (filterResult) {
          self.searchTokenList = filterResult as Array<ISearchTokenListItem>;
        }
        return;
      }
      // @ts-ignore
      const res = await self.fetchERC20Detail({
        address: val,
        symbol: '',
        tag: '',
        dsymbol: '',
      });
      if (!res?.dsymbol) {
        res.dsymbol = res.symbol;
      }
      res.tag = 'uToken';
      const fined = find(self.originalList, (o: ITokenListItem) => o.address === res.address);
      // @ts-ignore
      self.searchTokenList.push(fined || res);
    }, 300),
    selectToken(item) {
      this.$emit('select-token', {
        data: item,
        symbol: this.symbol,
      });
      this.dialogVisibleSelectToken = false;
    },
    async fetchBalanceOfDefaultList() {
      // const [...rest] = this.originalList;
      // const result = (
      //   await Promise.all(rest.map((item) => this.fetchERC20Detail(item)))
      // ).map(({ balance, ...res }) => ({ ...res, balance: balance.toString() }));
      // this.originalList = [...result];
      // this.originalList = sortBy(this.originalList, 'balance').reverse();
    },
    getERC20(_address) {
      return CommonERC20.attach(_address).connect(getProvider()!.getSigner());
    },
    async fetchERC20Detail(item) {
      // eslint-disable-next-line no-underscore-dangle
      const _address = item.address;
      const {
        name, symbol, totalSupply, decimals, balance, dBalance,
      } = await fetchERC20DetailInfo(_address, this.address);
      return {
        ...item,
        name,
        symbol,
        totalSupply,
        decimals,
        balance,
        dBalance,
      };
    },
  },
});
</script>

<style lang="scss" scoped>
.select-token-model-body {
  display: block;
}
.select-token-model-search {
  padding: 0 20px;
  margin-bottom: 14px;
}
.select-token-model-input {
  box-sizing: border-box;
  padding: 16px;
  -webkit-box-align: center;
  align-items: center;
  width: 100%;
  white-space: nowrap;
  background: none;
  outline: none;
  border-radius: 20px;
  color: rgb(255, 255, 255);
  border: 1px solid rgb(64, 68, 79);
  appearance: none;
  font-size: 18px;
  transition: border 100ms ease 0s;
  &:hover {
    border: 1px solid rgb(33, 114, 229);
    outline: none;
  }
}
.select-token-model-filter {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px;
  margin-bottom: 14px;
  &-text {
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    font-size: 14px;
    font-weight: 500;
  }
  &-button {
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    display: flex;
    -webkit-box-align: center;
    align-items: center;
    width: fit-content;
    padding: 8px;
    background-color: rgb(44, 47, 54);
    color: rgb(255, 255, 255);
    border-radius: 8px;
    user-select: none;
    cursor: pointer;
  }
}
.select-token-model-list {
  display: flex;
  flex-direction: column;
  border-top: 1px solid rgb(44, 47, 54);
  height: 500px;
  overflow-y: auto;
}
.common-bases {
  display: grid;
  grid-auto-rows: auto;
  row-gap: 12px;
  padding: 0 20px;
  margin-bottom: 14px;
}
.cb-header {
  box-sizing: border-box;
  margin: 0px;
  min-width: 0px;
  width: 100%;
  display: flex;
  padding: 0px;
  -webkit-box-align: center;
  align-items: center;
  flex-wrap: wrap;
  .cbh-title {
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    font-weight: 500;
    font-size: 14px;
  }
}
.cb-item {
  box-sizing: border-box;
  min-width: 0px;
  width: 100%;
  display: flex;
  padding: 0px;
  -webkit-box-align: center;
  align-items: center;
  flex-wrap: wrap;
  margin: -4px;
  & > * {
    margin: 4px;
  }
}
.cbi-token {
  border: 1px solid rgb(64, 68, 79);
  border-radius: 10px;
  display: flex;
  padding: 6px;
  -webkit-box-align: center;
  align-items: center;

  &:hover {
    cursor: pointer;
    background-color: rgb(44, 47, 54);
  }

  .cbit-logo {
    width: 24px;
    height: 24px;
    box-shadow: rgba(0, 0, 0, 0.075) 0px 6px 10px;
    border-radius: 24px;
    margin-right: 8px;
  }
  .cbit-symbol {
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    font-weight: 500;
    font-size: 16px;
  }
  .cbit-symbol-help {
    width: 24px;
    height: 24px;
    margin-right: 8px;
  }
}
</style>
