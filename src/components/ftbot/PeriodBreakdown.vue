<script setup lang="ts">
import { useBotStore } from '@/stores/ftbotwrapper';
import type { TableField } from 'bootstrap-vue-next';

import { TimeSummaryOptions } from '@/types';

const botStore = useBotStore();
const settingsStore = useSettingsStore();

const props = defineProps<{
  multiBotView?: boolean;
}>();

const hasWeekly = computed(() => botStore.activeBot.botApiVersion >= 2.33 || props.multiBotView);

const periodicBreakdownSelections = computed(() => {
  const vals = [{ value: TimeSummaryOptions.daily, text: 'Days' }];
  if (hasWeekly.value) {
    vals.push({ value: TimeSummaryOptions.weekly, text: 'Weeks' });
    vals.push({ value: TimeSummaryOptions.monthly, text: 'Months' });
  }
  return vals;
});

const absRelSelections = ref([
  { value: 'abs_profit', text: 'Abs $' },
  { value: 'rel_profit', text: 'Rel %' },
]);

const selectedStats = computed(() => {
  if (props.multiBotView) {
    switch (settingsStore.timeProfitPeriod) {
      case TimeSummaryOptions.weekly:
        return botStore.allWeeklyStatsSelectedBots;
      case TimeSummaryOptions.monthly:
        return botStore.allMonthlyStatsSelectedBots;
      default:
        return botStore.allDailyStatsSelectedBots;
    }
  }

  switch (settingsStore.timeProfitPeriod) {
    case TimeSummaryOptions.weekly:
      return botStore.activeBot.weeklyStats;
    case TimeSummaryOptions.monthly:
      return botStore.activeBot.monthlyStats;
    default:
      return botStore.activeBot.dailyStats;
  }
});

const selectedStatsSorted = computed(() => {
  // Sorted version for chart
  return {
    ...selectedStats.value,
    data: selectedStats.value.data
      ? Object.values(selectedStats.value.data).sort((a, b) => (a.date > b.date ? 1 : -1))
      : [],
  };
});

const dailyFields = computed<TableField[]>(() => {
  const res: TableField[] = [
    { key: 'date', label: 'Day' },
    {
      key: 'abs_profit',
      label: 'Profit',
      formatter: (value: unknown) =>
        formatPrice(value as number, botStore.activeBot.stakeCurrencyDecimals),
    },
    {
      key: 'fiat_value',
      label: `In ${botStore.activeBot.dailyStats.fiat_display_currency}`,
      formatter: (value: unknown) => formatPrice(value as number, 2),
    },
    { key: 'trade_count', label: 'Trades' },
  ];
  if (botStore.activeBot.botApiVersion >= 2.16)
    res.push({
      key: 'rel_profit',
      label: 'Profit%',
      formatter: (value: unknown) => formatPercent(value as number, 2),
    });
  return res;
});

function refreshSummary() {
  if (props.multiBotView) {
    botStore.allGetTimeSummary(settingsStore.timeProfitPeriod);
  } else {
    botStore.activeBot.getTimeSummary(settingsStore.timeProfitPeriod);
  }
}

onMounted(() => {
  refreshSummary();
});
</script>

<template>
  <div class="d-flex flex-column h-100">
    <div v-if="!props.multiBotView" class="mb-2">
      <h3 class="me-auto d-inline">{{ hasWeekly ? 'Period' : 'Daily' }} Breakdown</h3>
      <BButton class="float-end" size="sm" @click="refreshSummary">
        <i-mdi-refresh />
      </BButton>
    </div>
    <div class="d-flex align-items-center justify-content-between">
      <BFormRadioGroup
        v-if="hasWeekly"
        id="order-direction"
        v-model="settingsStore.timeProfitPeriod"
        :options="periodicBreakdownSelections"
        name="radios-btn-default"
        size="sm"
        buttons
        style="min-width: 10em"
        button-variant="outline-primary"
        @change="refreshSummary"
      ></BFormRadioGroup>
      <BFormRadioGroup
        v-model="settingsStore.timeProfitPreference"
        name="radios-btn-select"
        size="sm"
        :options="absRelSelections"
        buttons
        button-variant="outline-primary"
      >
      </BFormRadioGroup>
    </div>

    <div class="ps-1">
      <TimePeriodChart
        v-if="selectedStats"
        :daily-stats="selectedStatsSorted"
        :show-title="false"
        :profit-col="settingsStore.timeProfitPreference"
      />
    </div>
    <div v-if="!props.multiBotView">
      <BTable class="table-sm" :items="selectedStats.data" :fields="dailyFields"> </BTable>
    </div>
  </div>
</template>
