<script setup lang="ts"></script>

<template>
  <div class="home">
    <div class="d-flex justify-content-center">
      <BotList />
    </div>
    <hr />
    <!-- <img alt="Freqtrade logo" src="../assets/freqtrade-logo.png" width="450px" class="my-5" /> -->
    <div title="CryptoGPT_Freqtrade logo" class="logo-svg my-5 mx-auto" />
    <div>
      <h1>Welcome to the CryptoGPT UI</h1>
    </div>
    <div>This page allows you to control your trading bot.</div>
    <br />
    <p>
      If you need any help, please refer to the
      <a href="https://www.freqtrade.io/en/latest/">CryptoGPT_Freqtrade Documentation</a>.
    </p>

    <p>Have fun - <i>wishes you the CryptoGPT_Freqtrade team</i></p>
  </div>
</template>

<style lang="scss" scoped>
.home {
  margin-top: 1.5em;
}
.logo-svg {
  -webkit-mask: url(../assets/freqtrade-logo-mask.png) no-repeat center;
  -webkit-mask-size: 240px 240px;
  mask: url(../assets/freqtrade-logo-mask.png) no-repeat center;
  mask-size: contain;
  width: 250px;
  height: 250px;
}
</style>
