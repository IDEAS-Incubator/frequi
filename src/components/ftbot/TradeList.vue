<script setup lang="ts">
import type { MultiDeletePayload, MultiForcesellPayload, Trade } from '@/types';

import { useBotStore } from '@/stores/ftbotwrapper';
import { useRouter } from 'vue-router';
import type { TableField, TableItem } from 'bootstrap-vue-next';

enum ModalReasons {
  removeTrade,
  forceExit,
  forceExitPartial,
  cancelOpenOrder,
}

const props = defineProps({
  trades: { required: true, type: Array as () => Array<Trade> },
  title: { default: 'Trades', type: String },
  stakeCurrency: { required: false, default: '', type: String },
  activeTrades: { default: false, type: Boolean },
  showFilter: { default: false, type: Boolean },
  multiBotView: { default: false, type: Boolean },
  emptyText: { default: 'No Trades to show.', type: String },
});
const botStore = useBotStore();
const router = useRouter();
const settingsStore = useSettingsStore();
const currentPage = ref(1);
const selectedItemIndex = ref();
const filterText = ref('');
const feTrade = ref<Trade>({} as Trade);
const perPage = props.activeTrades ? 200 : 15;
const tradesTable = ref<HTMLFormElement>();
const forceExitVisible = ref(false);
const removeTradeVisible = ref(false);
const confirmExitText = ref('');
const confirmExitValue = ref<ModalReasons | null>(null);

const increasePosition = ref({ visible: false, trade: {} as Trade });
function formatPriceWithDecimals(price) {
  return formatPrice(price, botStore.activeBot.stakeCurrencyDecimals);
}
const rows = computed(() => {
  return props.trades.length;
});

// This using "TableField[]" below causes
// Error: Debug Failure. No error for last overload signature
const tableFields = ref<any[]>([]);

onMounted(() => {
  const openFields: TableField[] = [{ key: 'actions' }];
  const closedFields: TableField[] = [
    { key: 'close_timestamp', label: 'Close date' },
    { key: 'exit_reason', label: 'Close Reason' },
  ];
  const stakeAmountCol: TableField = props.activeTrades
    ? {
        key: 'stake_amount',
        label: 'Stake amount',
      }
    : {
        key: 'max_stake_amount',
        label: 'Total stake amount',
      };

  tableFields.value = [
    { key: 'trade_id', label: 'ID' },
    { key: 'pair', label: 'Pair' },
    { key: 'amount', label: 'Amount' },
    stakeAmountCol,
    {
      key: 'open_rate',
      label: 'Open rate',
      formatter: (value: unknown) => formatPrice(value as number),
    },
    {
      key: props.activeTrades ? 'current_rate' : 'close_rate',
      label: props.activeTrades ? 'Current rate' : 'Close rate',
      formatter: (value: unknown) => formatPrice(value as number),
    },
    {
      key: 'profit',
      label: props.activeTrades ? 'Current profit %' : 'Profit %',
      formatter: (value: unknown, key?: string, item?: unknown) => {
        if (!item) {
          return '';
        }
        const typedItem = item as Trade;
        const percent = formatPercent(typedItem.profit_ratio, 2);
        return `${percent} ${`(${formatPriceWithDecimals(typedItem.profit_abs)})`}`;
      },
    },
    { key: 'open_timestamp', label: 'Open date' },
    ...(props.activeTrades ? openFields : closedFields),
  ];
  if (props.multiBotView) {
    tableFields.value.unshift({ key: 'botName', label: 'Bot' });
  }
});

const feOrderType = ref<string | undefined>(undefined);
function forceExitHandler(item: Trade, ordertype: string | undefined = undefined) {
  feTrade.value = item;
  confirmExitValue.value = ModalReasons.forceExit;
  confirmExitText.value = `Really exit trade ${item.trade_id} (Pair ${item.pair}) using ${ordertype} Order?`;
  feOrderType.value = ordertype;
  if (settingsStore.confirmDialog === true) {
    removeTradeVisible.value = true;
  } else {
    forceExitExecuter();
  }
}

function forceExitExecuter() {
  if (confirmExitValue.value === ModalReasons.removeTrade) {
    const payload: MultiDeletePayload = {
      tradeid: String(feTrade.value.trade_id),
      botId: feTrade.value.botId,
    };
    botStore.deleteTradeMulti(payload).catch((error) => console.log(error.response));
  }
  if (confirmExitValue.value === ModalReasons.forceExit) {
    const payload: MultiForcesellPayload = {
      tradeid: String(feTrade.value.trade_id),
      botId: feTrade.value.botId,
    };
    if (feOrderType.value) {
      payload.ordertype = feOrderType.value;
    }
    botStore
      .forceSellMulti(payload)
      .then((xxx) => console.log(xxx))
      .catch((error) => console.log(error.response));
  }
  if (confirmExitValue.value === ModalReasons.cancelOpenOrder) {
    const payload: MultiDeletePayload = {
      tradeid: String(feTrade.value.trade_id),
      botId: feTrade.value.botId,
    };
    botStore.cancelOpenOrderMulti(payload);
  }

  feOrderType.value = undefined;
  removeTradeVisible.value = false;
}

function removeTradeHandler(item: Trade) {
  confirmExitText.value = `Really delete trade ${item.trade_id} (Pair ${item.pair})?`;
  confirmExitValue.value = ModalReasons.removeTrade;
  feTrade.value = item;
  removeTradeVisible.value = true;
}

function forceExitPartialHandler(item: Trade) {
  feTrade.value = item;
  forceExitVisible.value = true;
}

function cancelOpenOrderHandler(item: Trade) {
  confirmExitText.value = `Cancel open order for trade ${item.trade_id} (Pair ${item.pair})?`;
  feTrade.value = item;
  confirmExitValue.value = ModalReasons.cancelOpenOrder;
  removeTradeVisible.value = true;
}

function reloadTradeHandler(item: Trade) {
  botStore.reloadTradeMulti({ tradeid: String(item.trade_id), botId: item.botId });
}

function handleForceEntry(item: Trade) {
  increasePosition.value.trade = item;
  increasePosition.value.visible = true;
}

function handleContextMenuEvent(item, index, event) {
  // stop browser context menu from appearing
  if (!props.activeTrades) {
    return;
  }
  event.preventDefault();
  // log the selected item to the console
  console.log(item);
}

const onRowClicked = (item) => {
  if (props.multiBotView && botStore.selectedBot !== item.botId) {
    // Multibotview - on click switch to the bot trade view
    botStore.selectBot(item.botId);
  }
  if (item && item.trade_id !== botStore.activeBot.detailTradeId) {
    botStore.activeBot.setDetailTrade(item);
    if (props.multiBotView) {
      router.push({ name: 'CryptoGPT_Freqtrade Trading' });
    }
  } else {
    botStore.activeBot.setDetailTrade(null);
  }
};

const onRowSelected = () => {
  if (botStore.activeBot.detailTradeId) {
    const itemIndex = props.trades.findIndex(
      (v) => v.trade_id === botStore.activeBot.detailTradeId,
    );
    if (itemIndex >= 0) {
      tradesTable.value?.selectRow(itemIndex);
    } else {
      console.log(`Unsetting item for tradeid ${selectedItemIndex.value}`);
      selectedItemIndex.value = undefined;
    }
  }
};

watch(
  () => botStore.activeBot.detailTradeId,
  (val) => {
    const index = props.trades.findIndex((v) => v.trade_id === val);
    // Unselect when another tradeTable is selected!
    if (index < 0) {
      tradesTable.value?.clearSelected();
    }
  },
);
</script>

<template>
  <div class="h-100 overflow-auto w-100">
    <BTable
      ref="tradesTable"
      small
      hover
      stacked="md"
      :items="
        trades.filter(
          (t) =>
            t.pair.toLowerCase().includes(filterText.toLowerCase()) ||
            t.exit_reason?.toLowerCase().includes(filterText.toLowerCase()) ||
            t.enter_tag?.toLowerCase().includes(filterText.toLowerCase()),
        ) as unknown as TableItem[]
      "
      :fields="tableFields"
      show-empty
      :empty-text="emptyText"
      :per-page="perPage"
      :current-page="currentPage"
      primary-key="botTradeId"
      selectable
      :select-head="false"
      select-mode="single"
      @row-contextmenu="handleContextMenuEvent"
      @row-clicked="onRowClicked"
      @row-selected="onRowSelected"
    >
      <template #cell(actions)="{ index, item }">
        <TradeActionsPopover
          :id="index"
          :enable-force-entry="botStore.activeBot.botState.force_entry_enable"
          :trade="item as unknown as Trade"
          :bot-api-version="botStore.activeBot.botApiVersion"
          @delete-trade="removeTradeHandler(item as unknown as Trade)"
          @force-exit="forceExitHandler"
          @force-exit-partial="forceExitPartialHandler"
          @cancel-open-order="cancelOpenOrderHandler"
          @reload-trade="reloadTradeHandler"
          @force-entry="handleForceEntry"
        />
      </template>
      <template #cell(pair)="row">
        <span>
          {{ `${row.item.pair}${row.item.open_order_id || row.item.has_open_orders ? '*' : ''}` }}
        </span>
      </template>
      <template #cell(trade_id)="row">
        {{ row.item.trade_id }}
        {{
          botStore.activeBot.botApiVersion > 2.0 && row.item.trading_mode !== 'spot'
            ? '| ' + (row.item.is_short ? 'Short' : 'Long')
            : ''
        }}
      </template>
      <template #cell(stake_amount)="row">
        {{ formatPriceWithDecimals(row.item.stake_amount) }}
        {{ row.item.trading_mode !== 'spot' ? `(${row.item.leverage}x)` : '' }}
      </template>
      <template #cell(profit)="row">
        <TradeProfit :trade="row.item as unknown as Trade" />
      </template>
      <template #cell(open_timestamp)="row">
        <DateTimeTZ :date="(row.item as unknown as Trade).open_timestamp" />
      </template>
      <template #cell(close_timestamp)="row">
        <DateTimeTZ :date="(row.item as unknown as Trade).close_timestamp ?? 0" />
      </template>
    </BTable>
    <div class="w-100 d-flex justify-content-between">
      <BPagination
        v-if="!activeTrades"
        v-model="currentPage"
        :total-rows="rows"
        :per-page="perPage"
        aria-controls="my-table"
      ></BPagination>
      <BFormGroup v-if="showFilter" label-for="trade-filter">
        <BFormInput id="trade-filter" v-model="filterText" type="text" placeholder="Filter" />
      </BFormGroup>
    </div>
    <ForceExitForm
      v-if="activeTrades"
      v-model="forceExitVisible"
      :trade="feTrade"
      :stake-currency-decimals="botStore.activeBot.botState.stake_currency_decimals ?? 3"
    />
    <ForceEntryForm
      v-model="increasePosition.visible"
      :pair="increasePosition.trade?.pair"
      position-increase
    />

    <BModal v-model="removeTradeVisible" title="Exit trade" @ok="forceExitExecuter">
      {{ confirmExitText }}
    </BModal>
  </div>
</template>

<style lang="scss" scoped>
.card-body {
  padding: 0 0.2em;
}
.table-sm {
  font-size: $fontsize-small;
}
.btn-xs {
  padding: 0.1rem 0.25rem;
  font-size: 0.75rem;
}
</style>
