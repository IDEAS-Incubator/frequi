<script setup lang="ts">
import type { AuthPayload, AuthStorageWithBotId } from '@/types';

import { useBotStore } from '@/stores/ftbotwrapper';
import { useRoute, useRouter } from 'vue-router';
import axios from 'axios';

const props = defineProps({
  inModal: { default: false, type: Boolean },
  existingAuth: { default: null, required: false, type: Object as () => AuthStorageWithBotId },
});
const emit = defineEmits<{ loginResult: [value: boolean] }>();

const defaultURL = window.location.origin || 'http://localhost:3000';

const router = useRouter();
const route = useRoute();
const botStore = useBotStore();

const nameState = ref<boolean>();
const pwdState = ref<boolean>();
const urlState = ref<boolean>();
const errorMessage = ref<string>('');
const errorMessageCORS = ref<boolean>(false);
const formRef = ref<HTMLFormElement>();
const botEdit = ref<boolean>(false);
const auth = ref<AuthPayload>({
  botName: '',
  url: defaultURL,
  username: '',
  password: '',
});

const emitLoginResult = (value: boolean) => {
  emit('loginResult', value);
};

const urlDuplicate = computed<boolean>(() => {
  const bots = Object.values(botStore.availableBots).find((bot) => bot.botUrl === auth.value.url);
  return !botEdit.value && bots !== undefined;
});

const checkFormValidity = () => {
  const valid = formRef.value?.checkValidity();
  nameState.value = valid || auth.value.username !== '';
  pwdState.value = valid || auth.value.password !== '';
  urlState.value = valid || auth.value.url !== '';
  return valid;
};

const resetLogin = () => {
  auth.value.botName = '';
  auth.value.url = defaultURL;
  auth.value.username = '';
  auth.value.password = '';
  nameState.value = undefined;
  pwdState.value = undefined;
  urlState.value = undefined;
  errorMessage.value = '';
  botEdit.value = false;
};

const handleReset = (evt) => {
  evt.preventDefault();
  resetLogin();
};
const handleSubmit = async () => {
  // Exit when the form isn't valid
  if (!checkFormValidity()) {
    return;
  }
  errorMessage.value = '';
  // Push the name to submitted names
  try {
    const botId = botEdit.value ? props.existingAuth.botId : botStore.nextBotId;
    const { login } = useLoginInfo(botId);
    await login(auth.value);
    if (botEdit.value) {
      // Bot editing ...
      botStore.botStores[botId].isBotLoggedIn = true;
      botStore.botStores[botId].isBotOnline = true;
      // botStore.allRefreshFull();
      emitLoginResult(true);
    } else {
      // Add new bot
      const sortId = Object.keys(botStore.availableBots).length + 1;
      botStore.addBot({
        botName: auth.value.botName,
        botId,
        botUrl: auth.value.url,
        sortId: sortId,
      });
      // switch to newly added bot
      botStore.selectBot(botId);
      emitLoginResult(true);
      botStore.allRefreshFull();
    }

    if (props.inModal === false) {
      if (typeof route?.query.redirect === 'string') {
        const resolved = router.resolve({ path: route.query.redirect });
        if (resolved.name === '404') {
          router.push('/');
        } else {
          router.push(resolved.path);
        }
      } else {
        router.push('/');
      }
    }
  } catch (error) {
    errorMessageCORS.value = false;
    // this.nameState = false;
    console.error(error);
    if (axios.isAxiosError(error) && error.response && error.response.status === 401) {
      nameState.value = false;
      pwdState.value = false;
      errorMessage.value = 'Connected to bot, however Login failed, Username or Password wrong.';
    } else {
      urlState.value = false;
      errorMessage.value = `Login failed.
Please verify that the bot is running, the Bot API is enabled and the URL is reachable.
You can verify this by navigating to ${auth.value.url}/api/v1/ping to make sure the bot API is reachable`;
      if (auth.value.url !== window.location.origin) {
        errorMessageCORS.value = true;
      }
    }
    console.error(errorMessage.value);
    emitLoginResult(false);
  }
};

const handleOk = (evt) => {
  evt.preventDefault();
  handleSubmit();
};
const reset = () => {
  resetLogin();
  console.log('reset ', props.existingAuth);
  if (props.existingAuth) {
    botEdit.value = true;
    auth.value.botName = props.existingAuth.botName;
    auth.value.url = props.existingAuth.apiUrl;
    auth.value.username = props.existingAuth.username ?? '';
  }
};

defineExpose({
  handleSubmit,
  reset,
});
</script>

<template>
  <div>
    <form ref="formRef" novalidate @submit.stop.prevent="handleSubmit" @reset="handleReset">
      <BFormGroup label="Bot Name" label-for="name-input">
        <BFormInput
          id="name-input"
          v-model="auth.botName"
          placeholder="Bot Name"
          @keydown.enter="handleOk"
        ></BFormInput>
      </BFormGroup>
      <BFormGroup
        :state="urlState"
        label="API Url"
        label-for="url-input"
        invalid-feedback="API Url required"
      >
        <BFormInput
          id="url-input"
          v-model="auth.url"
          required
          trim
          :state="urlState"
          @keydown.enter="handleOk"
        ></BFormInput>
        <BAlert
          v-if="urlDuplicate"
          class="mt-2 p-1 alert-wrap"
          :model-value="true"
          variant="warning"
        >
          This URL is already in use by another bot.
        </BAlert>
      </BFormGroup>
      <BFormGroup
        :state="nameState"
        label="Username"
        label-for="username-input"
        invalid-feedback="Name and Password are required."
      >
        <BFormInput
          id="username-input"
          v-model="auth.username"
          required
          placeholder="CryptoGPT_Freqtrader"
          :state="nameState"
          @keydown.enter="handleOk"
        ></BFormInput>
      </BFormGroup>
      <BFormGroup
        label="Password"
        label-for="password-input"
        invalid-feedback="Invalid Password"
        :state="pwdState"
      >
        <BFormInput
          id="password-input"
          v-model="auth.password"
          required
          type="password"
          :state="pwdState"
          @keydown.enter="handleOk"
        ></BFormInput>
      </BFormGroup>
      <div>
        <BAlert v-if="errorMessage" class="alert-wrap" :model-value="true" variant="warning">
          {{ errorMessage }}
          <br />
          <span v-if="errorMessageCORS"
            >Please also check your bot's CORS configuration:
            <a href="https://www.freqtrade.io/en/latest/rest-api/#cors"
              >CryptoGPT_Freqtrade CORS documentation</a
            ></span
          >
        </BAlert>
      </div>
      <div v-if="inModal === false" class="float-end">
        <BButton class="me-2" type="reset" variant="danger">Reset</BButton>
        <BButton type="submit" variant="primary">Submit</BButton>
      </div>
    </form>
  </div>
</template>

<style scoped lang="scss">
.alert-wrap {
  white-space: pre-wrap;
}
</style>
