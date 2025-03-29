<script setup lang="ts">
import Favico from 'favico.js';

import { OpenTradeVizOptions, useSettingsStore } from '@/stores/settings';
import { useLayoutStore } from '@/stores/layout';
import { useBotStore } from '@/stores/ftbotwrapper';
import { useRoute } from 'vue-router';

const botStore = useBotStore();

const settingsStore = useSettingsStore();
const layoutStore = useLayoutStore();
const route = useRoute();
const router = useRouter();
const favicon = ref<Favico | undefined>(undefined);
const pingInterval = ref<number>();

async function clickLogout() {
  botStore.removeBot(botStore.selectedBot);
  // TODO: This should be per bot
  await router.push('/');
}

const setOpenTradesAsPill = (tradeCount: number) => {
  if (!favicon.value) {
    favicon.value = new Favico({
      animation: 'none',
      // position: 'up',
      // fontStyle: 'normal',
      // bgColor: '#',
      // textColor: '#FFFFFF',
    });
  }
  if (tradeCount !== 0 && settingsStore.openTradesInTitle === 'showPill') {
    favicon.value.badge(tradeCount);
  } else {
    favicon.value.reset();
    console.log('reset');
  }
};
const resetDynamicLayout = (): void => {
  console.log(`resetLayout called for ${route?.fullPath}`);
  switch (route?.fullPath) {
    case '/trade':
      layoutStore.resetTradingLayout();
      break;
    case '/dashboard':
      layoutStore.resetDashboardLayout();
      break;
    default:
  }
};
const setTitle = () => {
  let title = 'freqUI';
  if (settingsStore.openTradesInTitle === OpenTradeVizOptions.asTitle) {
    title = `(${botStore.activeBotorUndefined?.openTradeCount}) ${title}`;
  }
  if (botStore.activeBotorUndefined?.botName) {
    title = `${title} - ${botStore.activeBotorUndefined?.botName}`;
  }
  document.title = title;
};

onBeforeUnmount(() => {
  if (pingInterval.value) {
    clearInterval(pingInterval.value);
  }
});

onMounted(async () => {
  await settingsStore.loadUIVersion();
  pingInterval.value = window.setInterval(botStore.pingAll, 60000);
});

settingsStore.$subscribe((_, state) => {
  const needsUpdate = settingsStore.openTradesInTitle !== state.openTradesInTitle;
  if (needsUpdate) {
    setTitle();
    setOpenTradesAsPill(botStore.activeBotorUndefined?.openTradeCount || 0);
  }
});

watch(
  () => botStore.activeBotorUndefined?.botName,
  () => setTitle(),
);
watch(
  () => botStore.activeBotorUndefined?.openTradeCount,
  () => {
    if (settingsStore.openTradesInTitle === OpenTradeVizOptions.showPill) {
      setOpenTradesAsPill(botStore.activeBotorUndefined?.openTradeCount ?? 0);
    } else if (settingsStore.openTradesInTitle === OpenTradeVizOptions.asTitle) {
      setTitle();
    }
  },
);
</script>

<template>
  <header>
    <BNavbar toggleable="md" dark variant="primary">
      <RouterLink class="navbar-brand" exact to="/">
        <img class="logo" src="@/assets/freqtrade-logo.png" alt="Home Logo" />
        <span class="navbar-brand-title d-sm-none d-lg-inline">CryptoGPT UI</span>
      </RouterLink>

      <!-- TODO: For XS breakpoint, this should be here...  -->
      <!-- <ReloadControl class="me-3" /> -->
      <BNavbarToggle target="nav-collapse"></BNavbarToggle>

      <BCollapse id="nav-collapse" class="text-center" is-nav>
        <BNavbarNav>
          <BNavItem v-if="!botStore.canRunBacktest" to="/trade">Trade</BNavItem>
          <BNavItem v-if="!botStore.canRunBacktest" to="/dashboard">Dashboard</BNavItem>
          <BNavItem to="/graph">Chart</BNavItem>
          <BNavItem to="/logs">Logs</BNavItem>
          <BNavItem v-if="botStore.canRunBacktest" to="/backtest">Backtest</BNavItem>
          <BNavItem v-if="botStore.isWebserverMode && botStore.activeBot.botApiVersion >= 2.41" to="/download_data">
            Download Data</BNavItem>
          <BNavItem v-if="
            (botStore.activeBot?.isWebserverMode ?? false) &&
            botStore.activeBot.botApiVersion >= 2.3
          " to="/pairlist_config">Pairlist Config</BNavItem>
          
          <BNavItem href="http://54.215.73.141/" target="_blank" rel="noopener noreferrer">
            Market Sentiment
          </BNavItem>

          <ThemeSelect />
        </BNavbarNav>

        <!-- Right aligned nav items -->
        <BNavbarNav class="ms-auto" menu-class="w-100">
          <!-- TODO This should show outside of the dropdown in XS mode -->
          <div v-if="!settingsStore.confirmDialog" class="my-auto me-5"
            title="Confirm dialog deactivated, Forced exits will be executed immediately. Be careful."
            style="color: yellow">
            <i-mdi-run-fast />
            <i-mdi-alert />
          </div>
          <div class="d-flex justify-content-between">
            <BDropdown v-if="botStore.botCount > 1" size="sm" class="m-1" no-caret variant="info"
              toggle-class="d-flex align-items-center " menu-class="my-0 py-0">
              <template #button-content>
                <BotEntry :bot="botStore.selectedBotObj" :no-buttons="true" />
              </template>
              <BotList :small="true" />
            </BDropdown>
            <ReloadControl class="me-3" title="Confirm Dialog deactivated." />
          </div>
          <li class="d-none d-md-flex flex-md-wrap flex-lg-nowrap align-items-center nav-item text-secondary me-2">
            <BNavText class="small me-2">
              {{
                (botStore.activeBotorUndefined && botStore.activeBotorUndefined.botName) ||
                'No bot selected'
              }}
            </BNavText>
            <BNavText v-if="botStore.botCount === 1">
              {{
                botStore.activeBotorUndefined && botStore.activeBotorUndefined.isBotOnline
                  ? 'Online'
                  : 'Offline'
              }}
            </BNavText>
          </li>
          <li v-if="botStore.hasBots" class="nav-item">
            <!-- Hide dropdown on xs, instead show below  -->
            <BNavItemDropdown id="avatar-drop" right auto-close class="d-none d-md-block">
              <template #button-content>
                <BAvatar size="2em" button>FT</BAvatar>
              </template>
              <span class="ps-3">V: {{ settingsStore.uiVersion }}</span>
              <!-- Link active-class to non-existant class to avoid it getting the "light" active color -->
              <BDropdownItem active-class="non-existant" to="/settings">Settings</BDropdownItem>
              <div class="ps-3">
                <BFormCheckbox v-model="layoutStore.layoutLocked">Lock layout</BFormCheckbox>
              </div>
              <BDropdownItem @click="resetDynamicLayout">Reset Layout</BDropdownItem>
              <template v-if="botStore.botCount === 1">
                <BDropdownDivider />
                <BDropdownItem active-class="non-existant" @click="clickLogout()">
                  <i-mdi-logout class="me-1" />
                  Sign Out
                </BDropdownItem>
              </template>
            </BNavItemDropdown>
            <div class="d-block d-md-none">
              <!-- Visible only on XS -->
              <!-- <li class="nav-item text-secondary ms-2 d-sm-none d-flex justify-content-between">
                <div class="d-flex">
                  <b-nav-text class="small me-2">
                    {{
                      (botStore.activeBotorUndefined && botStore.activeBotorUndefined.botName) ||
                      'No bot selected'
                    }}
                  </b-nav-text>
                </div>
              </li> -->
              <BNavItem class="py-0" to="/settings" title="Settings">
                Settings <i-mdi-cog class="ms-auto" />
              </BNavItem>
              <BNavItem v-if="botStore.botCount === 1" class="nav-link navbar-nav" to="/" @click="clickLogout()">Sign
                Out
              </BNavItem>
            </div>
          </li>
          <li v-else>
            <!-- should open Modal window! -->
            <LoginModal v-if="route?.path !== '/login'" />
          </li>
        </BNavbarNav>
      </BCollapse>
    </BNavbar>
  </header>
</template>

<style lang="scss" scoped>
.logo {
  vertical-align: middle;
  height: 35px;
  transition: transform 0.2s ease;
}

.logo:hover {
  transform: scale(1.05);
}

.dropdown-toggle::after {
  display: none;
}

.navbar-brand-title {
  padding-left: 0.8em;
  font-weight: 600;
  font-size: 1.2rem;
  letter-spacing: 0.5px;
}

.navbar {
  padding: 0.4rem 1rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background: linear-gradient(135deg, #2c3e50 0%, #3498db 100%) !important;
}

.nav-link {
  padding: 0.5rem 1rem;
  font-weight: 500;
  transition: all 0.2s ease;
  border-radius: 4px;
  margin: 0 0.2rem;
}

.nav-link:hover {
  background-color: rgba(255, 255, 255, 0.1);
  transform: translateY(-1px);
}

.nav-item {
  margin: 0 0.2rem;
}

:deep(.b-navbar-nav) {
  gap: 0.5rem;
}

:deep(.b-avatar) {
  border: 2px solid rgba(255, 255, 255, 0.2);
  transition: all 0.2s ease;
}

:deep(.b-avatar:hover) {
  border-color: rgba(255, 255, 255, 0.4);
  transform: scale(1.05);
}

:deep(.dropdown-menu) {
  border: none;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  padding: 0.5rem;
}

:deep(.dropdown-item) {
  padding: 0.5rem 1rem;
  border-radius: 4px;
  transition: all 0.2s ease;
}

:deep(.dropdown-item:hover) {
  background-color: rgba(52, 152, 219, 0.1);
  transform: translateX(4px);
}

:deep(.navbar-toggler) {
  border: none;
  padding: 0.5rem;
}

:deep(.navbar-toggler:focus) {
  box-shadow: none;
}

:deep(.navbar-toggler-icon) {
  background-image: none;
  position: relative;
  width: 24px;
  height: 2px;
  background-color: white;
  transition: all 0.3s ease;
}

:deep(.navbar-toggler-icon::before),
:deep(.navbar-toggler-icon::after) {
  content: '';
  position: absolute;
  width: 24px;
  height: 2px;
  background-color: white;
  transition: all 0.3s ease;
}

:deep(.navbar-toggler-icon::before) {
  top: -6px;
}

:deep(.navbar-toggler-icon::after) {
  bottom: -6px;
}
</style>
