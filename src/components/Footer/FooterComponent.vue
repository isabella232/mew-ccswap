<template>
  <div class="component--footer py-2">
    <v-container class="text-center">
      <img :src="mewIconLight" alt="MEW" height="25" />
      <div>©{{ presentDate }} MyEtherWallet Inc. All rights reserved</div>

      <div class="font-weight-regular mb-3">
        Need Ethereum wallet? Get one from
        <a href="https://www.myetherwallet.com/wallet/create" target="_blank">
          <span class="text--primary font-weight-bold">
            myetherwallet.com
          </span>
        </a>
      </div>

      <v-btn
        @click="openInNewTab('https://www.myetherwallet.com/wallet/create')"
        size="small"
      >
        Get Ethereum wallet
      </v-btn>
    </v-container>
  </div>
</template>

<script setup lang="ts">
import mewIconLight from "@/assets/images/icon-mew-logo-light.svg";

const openInNewTab = (url: string) => {
  const newWindow = window.open(url, "_blank");
  newWindow?.focus();
};
const presentDate = new Date().getFullYear();
</script>

<style lang="scss" scoped>
.component--footer {
  background-color: #05c0a5;
  position: relative;
  z-index: 1;
}
</style>
