<script setup>
import { computed, ref, watch, nextTick, onMounted } from "vue";
import { state, CT } from "../state.js";
import LeftRail from "../components/LeftRail.vue";
import ScreenStage from "../components/ScreenStage.vue";
import StageMessage from "../components/StageMessage.vue";
import CalendarEvent from "../components/CalendarEvent.vue";
import CalendarMonth from "../components/CalendarMonth.vue";
const rail = ref(null),
  list = ref(null),
  kept = ref([]);
const parts = computed(() => CT.parts(state.now));
const time = computed(() => CT.clockText(parts.value).time);
const days = computed(() => state.settings.calendarDays || 2);
const next = computed(() =>
  state.calendar.status === "ok"
    ? state.calendar.events.find(
        (e) =>
          !e.allDay &&
          e.start > state.now &&
          CT.dayNumber(e.start) === CT.dayNumber(state.now),
      )
    : null,
);
const sections = computed(() => {
  if (state.calendar.status !== "ok") return [];
  const today = CT.dayNumber(state.now);
  return Array.from({ length: days.value }, (_, i) => ({
    label:
      i === 0
        ? "Today"
        : i === 1
          ? "Tomorrow"
          : CT.DAYS[CT.parts(state.now + i * 86400000).day],
    events: state.calendar.events
      .filter((e) =>
        i === 0
          ? e.end > state.now && CT.dayNumber(e.start) <= today
          : CT.dayNumber(e.start) === today + i,
      )
      .sort((a, b) => b.allDay - a.allDay || a.start - b.start),
  })).filter((s) => s.events.length);
});
const message = computed(() => {
  const status = state.calendar.status;
  if (["denied", "restricted", "writeOnly"].includes(status))
    return {
      title: "Calendar access is off",
      detail:
        "Allow “Car Thing Helper” in System Settings → Privacy & Security → Calendars.",
    };
  if (["loading", "notDetermined"].includes(status))
    return { title: "Loading your calendar…" };
  if (status !== "ok")
    return { title: "Calendar unavailable", detail: "Trying again shortly." };
  if (!sections.value.length)
    return {
      title: "Nothing scheduled",
      detail:
        days.value > 1
          ? "The next " + days.value + " days are clear."
          : "The rest of the day is clear.",
    };
  return {};
});
// JS overrides the CSS display-size fallback only when the current time is too wide.
function fitClock() {
  const clock = rail.value && rail.value.titleElement;
  if (!clock) return;
  clock.style.fontSize = "";
  for (let size = 104; clock.scrollWidth > clock.clientWidth && size > 72; ) {
    size -= 8;
    clock.style.fontSize = size + "px";
  }
}
let generation = 0;
async function fitEvents() {
  const run = ++generation;
  kept.value = sections.value.map((s) => ({ ...s, shown: s.events.length }));
  await nextTick();
  // Today comes first: the last day gives up events until it's just its heading and a count,
  // then goes, and only once it's down to today do today's own events start to go.
  while (
    run === generation &&
    list.value &&
    list.value.scrollHeight > list.value.clientHeight
  ) {
    const last = kept.value[kept.value.length - 1];
    if (kept.value.length > 1) {
      if (last.shown > 0) last.shown--;
      else kept.value.pop();
    } else if (last.shown > 1) last.shown--;
    else break;
    await nextTick();
  }
}
// Separate sources, each compared on its own: one getter returning a fresh array would count
// as a change on every tick and rebuild the agenda each second rather than each minute.
watch(
  [
    () => state.calendar,
    () => state.settings,
    () => Math.floor(state.now / 60000),
    () => state.current,
  ],
  () => {
    nextTick(fitClock);
    fitEvents();
  },
);
// The calendar button, pressed on the calendar, swaps the agenda for the month and back.
const view = ref("agenda"); // agenda | month | day (a date tapped in the month)
const detail = ref(null); // an event tapped in the agenda or a day: its card, over the stage
const picked = ref(null); // that date's day number
const dayList = ref(null);
const dayShown = ref(0);
const dayEvents = computed(() =>
  state.calendar.status !== "ok" || picked.value == null
    ? []
    : state.calendar.events
        .filter(
          (e) =>
            CT.dayNumber(e.start) <= picked.value &&
            CT.dayNumber(Math.max(e.end - 1, e.start)) >= picked.value,
        )
        .sort((a, b) => b.allDay - a.allDay || a.start - b.start),
);
const dayLabel = computed(() => {
  if (picked.value == null) return "";
  const p = CT.parts(picked.value * 86400000, 0); // a day number is its own UTC midnight
  const today = CT.dayNumber(state.now);
  const name = CT.DAYS[p.day] + ", " + CT.MONTHS[p.month] + " " + p.date;
  return picked.value === today ? "Today · " + name : name;
});
// As in the agenda: events go until the rest fit, then "+N more".
async function fitDay() {
  dayShown.value = dayEvents.value.length;
  await nextTick();
  while (dayList.value && dayShown.value > 1 && dayList.value.scrollHeight > dayList.value.clientHeight) {
    dayShown.value--;
    await nextTick();
  }
}
watch([dayEvents, view], () => view.value === "day" && fitDay());
const dayName = (ms) => {
  const p = CT.parts(ms);
  return CT.DAYS[p.day].slice(0, 3) + ", " + CT.MONTHS[p.month].slice(0, 3) + " " + p.date;
};
const detailWhen = computed(() => {
  const e = detail.value;
  if (!e) return { date: "", time: "" };
  const last = Math.max(e.end - 1, e.start); // an all-day event ends at the midnight after it
  const multi = CT.dayNumber(last) > CT.dayNumber(e.start);
  if (e.allDay)
    return { date: dayName(e.start) + (multi ? " – " + dayName(last) : ""), time: "All day" };
  return {
    date: dayName(e.start) + (multi ? " – " + dayName(last) : ""),
    time: CT.timeText(e.start) + (e.end > e.start ? " – " + CT.timeText(e.end) : ""),
  };
});
function pickDate({ year, month, date }) {
  picked.value = Date.UTC(year, month, date) / 86400000;
  view.value = "day";
}
const screen = CT.screen("calendar");
screen.show = () => {
  nextTick(fitClock);
  fitEvents();
};
// When idle: the agenda or the month.
screen.showView = (v) => {
  detail.value = null;
  if (v === "agenda" || v === "month") view.value = v;
};
screen.reselect = (steps = 1) => {
  if (detail.value) { detail.value = null; return; } // the button closes an open card first
  if (steps % 2) view.value = view.value === "agenda" ? "month" : view.value === "day" ? "month" : "agenda";
};
onMounted(() => {
  fitClock();
  fitEvents();
  if (document.fonts)
    document.fonts.ready.then(() => {
      fitClock();
      fitEvents();
    });
});
</script>
<template>
  <section
    id="screen-calendar"
    class="screen fill flex"
    :class="{ active: state.current === 'calendar', leaving: state.leaving === 'calendar' }"
  >
    <LeftRail
      ref="rail"
      variant="time"
      :eyebrow="CT.DAYS[parts.day] + ' ' + (parts.month + 1) + '/' + parts.date"
      :title="time"
    >
      <template #lower>
        <Transition name="view"
          ><span v-if="view === 'month' || view === 'day'"
            ><template v-if="next"
              >{{ CT.timeText(next.start) }}
              <b class="primary medium">{{ next.title }}</b></template
            ><template v-else-if="state.calendar.status === 'ok'"
              >No events today</template
            ></span
          ></Transition
        >
      </template>
    </LeftRail>
    <ScreenStage class="bg-panel">
      <Transition name="view"
        ><CalendarMonth v-if="view === 'month'" class="stage-safe" :now="state.now" :events="state.calendar.status === 'ok' ? state.calendar.events : []" @pick="pickDate"
      /></Transition>
      <Transition name="view"><div v-if="view === 'day'" class="k-dayview stage-safe" @click="view = 'month'">
        <div ref="dayList" class="k-list overflow-hidden fill">
          <div class="k-day font-small medium">{{ dayLabel }}</div>
          <div v-if="!dayEvents.length" class="mt-16 font-small medium muted">Nothing scheduled</div>
          <CalendarEvent v-for="(event, i) in dayEvents.slice(0, dayShown)" :key="i" :event="event" :now="state.now" @click.stop="detail = event" />
          <div v-if="dayEvents.length > dayShown" class="mt-16 font-small medium muted">+{{ dayEvents.length - dayShown }} more</div>
        </div>
      </div></Transition>
      <Transition name="view"><div v-show="view === 'agenda'" ref="list" class="k-list overflow-hidden stage-safe">
        <template v-for="section in kept" :key="section.label">
          <div
            v-if="section.label"
            class="k-day font-small medium"
            :class="{ muted: section.label !== 'Today' }"
          >
            {{ section.label }}
          </div>
          <CalendarEvent
            v-for="(event, i) in section.events.slice(0, section.shown)"
            :key="i"
            :event="event"
            :now="state.now"
            @click="detail = event"
          />
          <div
            v-if="section.events.length > section.shown"
            class="mt-16 font-small medium muted"
          >
            {{
              section.shown
                ? "+" + (section.events.length - section.shown) + " more"
                : section.events.length +
                  (section.events.length === 1 ? " event" : " events")
            }}
          </div>
        </template>
      </div></Transition>
      <Transition name="view"><div v-if="detail" class="k-detail stage-safe flex-col" @click.stop="detail = null"><!-- tap it to close -->
        <div class="k-detail-cal flex items-center gap-8 font-small medium muted"><span class="k-dot flex-none" :style="{ background: detail.color || 'var(--muted)' }" /><span class="truncate">{{ detail.calendar }}</span></div>
        <h3 class="k-detail-title font-medium semibold">{{ detail.title || 'Untitled' }}</h3>
        <div class="font-small medium">{{ detailWhen.date }}</div>
        <div class="font-small secondary">{{ detailWhen.time }}</div>
        <div v-if="detail.location" class="k-detail-where font-small weight-light muted">{{ detail.location }}</div>
      </div></Transition>
      <Transition name="view"
        ><StageMessage
          v-if="view === 'agenda'"
          :title="message.title"
          :detail="message.detail"
      /></Transition>
    </ScreenStage>
  </section>
</template>
