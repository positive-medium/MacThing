<script setup>
import { computed } from "vue";
import { CT } from "../state.js";
// This month as a grid, Sunday first, with today circled (Figma 154:4958).
const props = defineProps({ now: Number, events: { type: Array, default: () => [] } });
const emit = defineEmits(['pick']);
const month = computed(() => {
  const p = CT.parts(props.now);
  const length = new Date(Date.UTC(p.year, p.month + 1, 0)).getUTCDate();
  const first = (((p.day - (p.date - 1)) % 7) + 7) % 7; // weekday of the 1st
  const cells = Array.from({ length: first }, () => null).concat(
    Array.from({ length }, (_, i) => i + 1),
  );
  while (cells.length % 7) cells.push(null); // so the last week keeps its columns
  const weeks = [];
  for (let i = 0; i < cells.length; i += 7) weeks.push(cells.slice(i, i + 7));
  return { name: CT.MONTHS[p.month], today: p.date, year: p.year, month: p.month, weeks };
});
// The dates this month with an event (one running over several days marks each of them).
const busy = computed(() => {
  const first = Date.UTC(month.value.year, month.value.month, 1) / 86400000;
  const set = new Set();
  for (const e of props.events) {
    const last = CT.dayNumber(Math.max(e.end - 1, e.start));
    for (let d = Math.max(CT.dayNumber(e.start), first); d <= last && d < first + 31; d++) set.add(d - first + 1);
  }
  return set;
});
</script>
<template>
  <div class="k-month flex-col">
    <div class="k-month-name font-small semibold">{{ month.name }}</div>
    <div class="k-week k-weekdays font-small semibold muted">
      <span v-for="(d, i) in CT.DAYS" :key="i">{{ d[0] }}</span>
    </div>
    <div
      v-for="(week, w) in month.weeks"
      :key="w"
      class="k-week font-small bold"
    >
      <span
        v-for="(date, i) in week"
        :key="i"
        :class="{ muted: i === 0 || i === 6, 'k-pick': date >= month.today }"
        @click="date >= month.today && emit('pick', { year: month.year, month: month.month, date })"
        ><svg v-if="date === month.today" class="k-today" viewBox="0 0 56 56">
          <circle cx="28" cy="28" r="28" />
          <text x="28" y="34" text-anchor="middle">{{ date }}</text>
          <circle v-if="busy.has(date)" class="k-today-dot" cx="28" cy="45" r="2.5" /></svg
        ><template v-else
          >{{ date }}<i v-if="busy.has(date)" class="k-mark" /></template></span
      >
    </div>
  </div>
</template>
