<template>
  <div
    @click="$emit('on-click')"
    :title="name"
    :class="{
      'c-select-token-item': true,
      'c-select-token-item--disabled': disabled
    }"
  >
    <div class="c-select-token-icon">
      <div class="c-select-token-icon" v-if="isYToken">
        <img src="@/assets/base/y3d.png" class="c-select-token-icon-image" alt="Token logo" />
      </div>
      <div class="c-select-token-icon" v-else-if="logo">
        <img :src="logo" class="c-select-token-icon-image" alt="Token logo" />
      </div>
      <div class="c-select-token-icon" v-else>
        <HelpCircleIcon class="c-select-token-icon" />
      </div>
    </div>
    <div class="c-select-token-name">
      <div class="c-select-token-name-text">{{ symbol }}</div>
    </div>
    <div class="c-select-token-balance">
      <div class="c-select-token-balance-text">{{ balance }}</div>
    </div>
  </div>
</template>
<script lang="ts">
import Vue from 'vue';
import { HelpCircleIcon } from 'vue-feather-icons';

export default Vue.extend({
  name: 'SelectTokenListItem',
  components: {
    HelpCircleIcon,
  },
  props: {
    logo: {
      type: String,
      default:
        'data:image/png;base64,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',
    },
    name: {
      type: String,
      default: '',
    },
    symbol: {
      type: String,
      default: '',
    },
    balance: {
      type: String,
      default: '0',
    },
    disabled: {
      type: Boolean,
      default: false,
    },
    isYToken: {
      type: Boolean,
      default: false,
    },
  },
});
</script>
<style lang="scss" scoped>
.c-select-token-item {
  display: grid;
  grid-template-columns: auto minmax(auto, 1fr) minmax(0px, 150px);
  gap: 16px;
  box-sizing: border-box;
  -webkit-box-align: center;
  align-items: center;
  -webkit-box-pack: justify;
  justify-content: space-between;
  min-width: 0px;
  width: 100%;
  height: 56px;
  margin: 0px;
  padding: 4px 20px;
  cursor: pointer;
  opacity: 1;
  &:hover {
    background-color: rgb(44, 47, 54);
  }
  &--disabled {
    pointer-events: none;
    opacity: 0.5;
  }
}
.c-select-token-icon {
  width: 24px;
  height: 24px;
  &-image {
    width: 24px;
    height: 24px;
  }
}
.c-select-token-name {
  display: flex;
  flex-direction: column;
  -webkit-box-pack: start;
  justify-content: flex-start;
  &-text {
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    font-weight: 500;
    font-size: 16px;
  }
}
.c-select-token-balance {
  box-sizing: border-box;
  margin: 0px;
  min-width: 0px;
  display: flex;
  padding: 0px;
  -webkit-box-align: center;
  align-items: center;
  width: 100%;
  &-text {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    box-sizing: border-box;
    margin: 0px;
    min-width: 0px;
    width: 100%;
    text-align: right;
  }
}
.token-edit {
  box-sizing: border-box;
  margin: 0px;
  min-width: 0px;
  display: flex;
  padding: 0px;
  -webkit-box-align: center;
  align-items: center;
  width: fit-content;
  color: rgb(33, 114, 229);
  font-size: 14px;
}
.te-name {
  box-sizing: border-box;
  margin: 0px;
  min-width: 0px;
  font-weight: 500;
  color: rgb(195, 197, 203);
}
.te-edit {
  border: none;
  text-decoration: none;
  background: none;
  cursor: pointer;
  color: rgb(33, 114, 229);
  font-weight: 500;
  user-select: none;
}
</style>