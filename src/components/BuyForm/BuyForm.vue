<template>
  <div class="component--buy-form elevated-box pa-3 pa-sm-6 pa-md-8">
    <!-- ============================================================================= -->
    <!-- Fiat amount -->
    <!-- ============================================================================= -->
    <div class="mb-10">
      <div class="d-flex align-center">
        <div class="heading-4 text-uppercase">Price</div>
        <div v-if="loading.fiatAmount" class="ml-2">
          <span class="h3 font-weight-regular mr-1 text--mew">Loading</span>
          <v-progress-circular
            :size="11"
            :width="2"
            indeterminate
          ></v-progress-circular>
        </div>
      </div>
      <!--
      <h4>
        ** Daily buy limit:
        <span class="font-weight-medium">USD $50 ~ $20,000</span>
      </h4>
      -->
      <div class="d-flex mt-2">
        <v-text-field
          type="number"
          v-model.number="form.fiatAmount"
          required
          dense
          @update:modelValue="debounce_getFiatForCrypto"
        ></v-text-field>
        <v-select
          style="max-width: 100px"
          v-model="form.fiatSelected"
          :items="fiatItems"
        ></v-select>
      </div>
    </div>

    <!-- ============================================================================= -->
    <!-- Crypto amount -->
    <!-- ============================================================================= -->
    <div class="mb-10">
      <div class="d-flex align-center">
        <div class="heading-4 text-uppercase">Amount</div>
        <div v-if="loading.cryptoAmount" class="ml-2">
          <span class="h3 font-weight-regular mr-1 text--mew">Loading</span>
          <v-progress-circular
            :size="11"
            :width="2"
            indeterminate
          ></v-progress-circular>
        </div>
      </div>
      <!--
      <h4>
        ** Daily buy limit:
        <span class="font-weight-medium">USD $50 ~ $20,000</span>
      </h4>
      -->
      <div class="d-flex mt-2">
        <v-text-field
          type="number"
          v-model.number="form.cryptoAmount"
          required
          dense
          @update:modelValue="debounce_getCryptoForFiat"
        ></v-text-field>
        <v-select
          style="max-width: 100px"
          v-model="form.cryptoSelected"
          :items="cryptoItems"
        ></v-select>
      </div>
    </div>

    <!-- ============================================================================= -->
    <!-- Wallet address -->
    <!-- ============================================================================= -->
    <div>
      <div class="d-sm-flex align-center justify-space-between mb-2">
        <div class="heading-4 text-uppercase mr-2">ETH Address</div>
        <a
          class="small d-block mt-n1 mt-sm-0"
          href="https://www.myetherwallet.com/wallet/create"
          target="_blank"
        >
          Don't have one?
        </a>
      </div>
      <v-text-field
        v-model="form.address"
        required
        dense
        :error-messages="form.addressErrorMsg"
        @keyup="verifyAddress"
      ></v-text-field>
    </div>

    <!-- ============================================================================= -->
    <!-- Google ReCaptcha v3 -->
    <!-- ============================================================================= -->
    <!-- <div class="d-flex align-center justify-center mt-3 mb-5">
      <ReCaptcha @token="onReCaptchaToken" />
    </div> -->

    <!-- ============================================================================= -->
    <!-- Buy/Rest button -->
    <!-- ============================================================================= -->
    <div v-if="!loading.processingBuyForm" class="pt-2 text-center">
      <div>
        <v-btn
          rounded="pill"
          :disabled="!isValidForm"
          min-height="60px"
          min-width="200px"
          color="#05C0A5"
          @click="submitForm"
        >
          <div class="text--white">Continue</div>
        </v-btn>
      </div>

      <!--
      <div>
        <v-btn class="my-3" @click="resetForms" variant="text" size="small">
          Clear
        </v-btn>
      </div>
      -->

      <div class="text--secondary mt-6">
        You will be redirected to the partner's site
      </div>
    </div>

    <div v-else class="text-center py-5">
      <v-progress-circular
        :size="70"
        :width="7"
        indeterminate
        color="#05c0a5"
      ></v-progress-circular>
      <div
        class="text-center font-weight-bold mt-3"
        style="line-height: 1.4rem"
      >
        Processing purchase....
      </div>
    </div>

    <!-- ============================================================================= -->
    <!-- Buy limit warning -->
    <!-- ============================================================================= -->
    <v-snackbar v-model="loading.showAlert" multi-line timeout="5000">
      <div class="text-center pa-3" style="max-width: 350px">
        <img :src="mewWalletImg" alt="MEW doggy" style="max-width: 80px" />
        <h3 class="text--white" v-if="loading.alertMessage === ''">
          Uh oh... Maximum daily crypto buy limit is between
          <span style="font-size: 1.2rem" class="text--white font-weight-bold">
            USD $50 ~ $20,000
          </span>
        </h3>
        <h3
          style="font-size: 1.2rem"
          class="text--white font-weight-bold"
          v-else
        >
          {{ loading.alertMessage }}
        </h3>

        <v-btn class="mt-3" @click="loading.showAlert = false" size="small">
          Close
        </v-btn>
      </div>
    </v-snackbar>

    <!-- ============================================================================= -->
    <!-- END -->
    <!-- ============================================================================= -->
  </div>
</template>

<script setup lang="ts">
import { computed, reactive, watch, onMounted } from "vue";
import BigNumber from "bignumber.js";
// import ReCaptcha from "@/components/recaptcha/ReCaptcha.vue";
import { supportedCrypto, supportedFiat, getSimplexQuote } from "./prices";
import { executeSimplexPayment } from "./order";
import { debounce, isObject } from "lodash";
import WAValidator from "multicoin-address-validator";
import mewWallet from "@/assets/images/icon-mew-wallet.png";
// import SubmitForm from "./SubmitForm.vue";

const mewWalletImg = mewWallet;
const defaultFiatValue = "0";
const apiDebounceTime = 1000;

onMounted(() => {
  // Load URL parameter value and verify crypto address
  loadUrlParameters();
  verifyAddress();

  // Get crypto amount based on current fiat amount
  getCryptoForFiat(true);
});

// data

// non-reactive
const fiatItems: string[] = supportedFiat;
const cryptoItems: string[] = supportedCrypto;

// reactive
const form = reactive({
  fiatAmount: defaultFiatValue,
  fiatSelected: "USD",
  cryptoAmount: "1",
  cryptoSelected: "ETH",
  address: "",
  addressErrorMsg: "",
  reCaptchaToken: "",
  addressError: false,
});

const loading = reactive({
  fiatAmount: false,
  cryptoAmount: false,
  showAlert: false,
  processingBuyForm: false,
  alertMessage: "",
});

// watchers
watch(
  () => form.cryptoSelected,
  () => {
    verifyAddress();
    getCryptoForFiat(false);
  }
);

watch(
  () => form.fiatSelected,
  () => {
    verifyAddress();
    getFiatForCrypto();
  }
);

const isValidForm = computed(() => {
  const fiatBn = new BigNumber(form.fiatAmount);
  const cryptoBn = new BigNumber(form.cryptoAmount);
  return (
    fiatBn.gt(0) &&
    cryptoBn.gt(0) &&
    form.fiatSelected &&
    form.cryptoSelected &&
    form.address &&
    !form.addressError &&
    form.addressErrorMsg === ""
  );
});

// methods
const loadUrlParameters = () => {
  const queryString = window.location.search;

  if (queryString) {
    const urlParams = new URLSearchParams(queryString);
    const queryCryptoAmount = urlParams.get("crypto_amount");
    const queryFiat = urlParams.get("fiat");
    const queryCrypto = urlParams.get("crypto");
    const queryTo = urlParams.get("to");
    form.fiatSelected = queryFiat ? queryFiat : "USD";
    form.fiatAmount = queryCryptoAmount ? queryCryptoAmount : "100";
    form.cryptoSelected = queryCrypto ? queryCrypto : "ETH";
    form.cryptoAmount = queryCryptoAmount ? queryCryptoAmount : "1";
    form.address = queryTo ? queryTo : "";
  }
};
// const onReCaptchaToken = (token: string): void => {
//   form.reCaptchaToken = token;
// };

const getCryptoForFiat = async (isLoading: boolean): Promise<void> => {
  loading.cryptoAmount = true;
  try {
    const response = await getSimplexQuote(
      form.fiatSelected,
      form.cryptoSelected,
      form.cryptoSelected,
      form.cryptoAmount
    );
    form.cryptoAmount = response.crypto_amount;
    form.fiatAmount = response.fiat_amount;
    loading.cryptoAmount = false;
  } catch (e: any) {
    loading.cryptoAmount = false;
    loading.showAlert = true;
    errorHandler(e);

    if (isLoading) {
      return resetForm();
    }
  }
};

const getFiatForCrypto = async (): Promise<void> => {
  loading.fiatAmount = true;
  try {
    const response = await getSimplexQuote(
      form.fiatSelected,
      form.cryptoSelected,
      form.fiatSelected,
      form.fiatAmount
    );
    form.cryptoAmount = response.crypto_amount;
    loading.fiatAmount = false;
  } catch (e: any) {
    loading.fiatAmount = false;
    loading.showAlert = true;
    errorHandler(e);
    getCryptoForFiat(false);
  }
};

const errorHandler = (e: any): void => {
  const value = new BigNumber(form.fiatAmount).gt(0);
  if (value) {
    const isErrorObj = isObject(e.response.data.error);
    if (isErrorObj) {
      // eslint-disable-next-line
      const hasErr = e.response.data.error.hasOwnProperty("errors");
      if (hasErr) {
        loading.alertMessage = e.response.data.error.errors[0].message;
      }
    } else {
      loading.alertMessage = e.response.data.error;
    }
  }
};

const resetForm = (): void => {
  form.fiatAmount = defaultFiatValue;
  form.fiatSelected = "USD";
  form.cryptoAmount = "0";
  form.cryptoSelected = "ETH";
  form.address = "";
  debounce_getCryptoForFiat();
};

const debounce_getCryptoForFiat = debounce(() => {
  getCryptoForFiat(false);
}, apiDebounceTime);
const debounce_getFiatForCrypto = debounce(() => {
  getFiatForCrypto();
}, apiDebounceTime);

const verifyAddress = (): void => {
  const valid = WAValidator.validate(form.address, form.cryptoSelected);
  if (valid) {
    form.addressErrorMsg = "";
    form.addressError = false;
  } else {
    if (!form.address) {
      form.addressErrorMsg = "";
    } else {
      form.addressErrorMsg = `Please provide a valid ${form.cryptoSelected} address`;
    }
  }
};

const submitForm = (): void => {
  loading.processingBuyForm = true;
  executeSimplexPayment(
    form.fiatSelected,
    form.cryptoSelected,
    form.fiatSelected,
    form.fiatAmount,
    form.address
  );
};
</script>

<style lang="scss">
.component--buy-form {
  // Fix v-select(currency select) height issue
  .v-select .v-field .v-field__input > input {
    height: 0;
    width: 0;
  }

  .v-field--appended {
    background-color: black;
    .v-select__selection-text {
      color: white;
    }

    .v-field__append-inner {
      .v-icon {
        opacity: 1;
        &::before {
          color: white;
        }
      }
    }
  }

  .v-field__field {
    padding: 0;
  }

  .v-field__input {
    padding-top: 0;
  }

  .v-select__selection-text {
    display: flex;
    align-items: center;
  }

  // Adjust (text field) prefix font size
  .v-messages__message {
    font-weight: 300;
    font-size: 0.9rem;
  }
  .v-text-field__prefix {
    font-size: 0.8rem;
  }
}
</style>
