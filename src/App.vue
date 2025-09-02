<template>
  <div class="layout">
    <header>
      <div id="current-time" aria-live="polite">{{ currentTimeText }}</div>
    </header>

    <aside>
      <h2>모의고사 시간표</h2>
      <ul id="schedule-list" class="schedule" aria-live="polite">
        <li
          v-for="w in scheduleWindows"
          :key="w.subject + w.start"
          class="item"
          :class="[{ active: isActive(w) }, { break: w.break }]"
          :data-subject="w.subject"
        >
          <div class="subject">{{ w.subject }}</div>
          <div class="time">{{ w.start }} – {{ w.end }}</div>
        </li>
      </ul>
    </aside>

    <main>
      <div class="status">
        <div id="current-subject" class="subject" :class="{ done: statusState === 'done' }" style="font-size: 80px;">
          {{ currentSubjectText }}
        </div>
        <div id="window-time" class="window" style="font-size: 70px; color: white;">{{ windowText }}</div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

// === 설정: 오늘 진행할 모의고사 시간표 ===
const SCHEDULE = [
  { subject: '국어', start: '08:40', end: '10:00' },
  { subject: '수학', start: '10:30', end: '12:10' },
  { subject: '점심', start: '12:00', end: '13:00', break: true },
  { subject: '영어', start: '13:10', end: '14:20' },
  { subject: '한국사', start: '14:50', end: '15:20' },
  { subject: '사회탐구', start: '15:35', end: '16:15' },
  { subject: '과학탐구', start: '16:30', end: '17:10' },
]

// === 유틸 ===
function todayAt(hhmm) {
  const [h, m] = hhmm.split(':').map(Number)
  const d = new Date()
  d.setHours(h, m, 0, 0)
  return d
}

// 윈도우(각 과목의 시간창)
const scheduleWindows = computed(() =>
  SCHEDULE.map((s) => ({
    ...s,
    startAt: todayAt(s.start),
    endAt: todayAt(s.end),
  }))
)

// 현재 시각(KST 표기)
const now = ref(new Date())
const clockFmt = new Intl.DateTimeFormat('ko-KR', {
  timeZone: 'Asia/Seoul',
  hour: '2-digit',
  minute: '2-digit',
  second: '2-digit',
})
const currentTimeText = computed(() => clockFmt.format(now.value))

// 상태 계산
function computeStatus(n) {
  const wins = scheduleWindows.value
  for (const w of wins) {
    if (n >= w.startAt && n < w.endAt) {
      return { state: 'in', window: w, remainMs: w.endAt - n }
    }
  }
  if (n < wins[0].startAt) {
    return { state: 'pre', next: wins[0], untilMs: wins[0].startAt - n }
  }
  for (let i = 0; i < wins.length - 1; i++) {
    const a = wins[i], b = wins[i + 1]
    if (n >= a.endAt && n < b.startAt) {
      return { state: 'gap', next: b, untilMs: b.startAt - n }
    }
  }
  return { state: 'done' }
}

const status = computed(() => computeStatus(now.value))
const statusState = computed(() => status.value.state)

const currentSubjectText = computed(() => {
  const st = status.value
  if (st.state === 'in') return st.window.break ? '휴식' : `${st.window.subject}`
  if (st.state === 'pre' || st.state === 'gap') return `다음: ${st.next.subject}`
  return '오늘 일정 종료'
})

const windowText = computed(() => {
  const st = status.value
  if (st.state === 'in') return `${st.window.start} – ${st.window.end}`
  if (st.state === 'pre' || st.state === 'gap') return `${st.next.start} – ${st.next.end}`
  return '모든 과목 완료'
})

function isActive(w) {
  const st = status.value
  return st.state === 'in' && st.window.subject === w.subject
}

let intervalId
onMounted(() => {
  intervalId = setInterval(() => {
    now.value = new Date()
  }, 1000)
})

onBeforeUnmount(() => {
  clearInterval(intervalId)
})
</script>

<style>
:root {
  --bg: #0a0a0a;
  --panel: #111111;
  --text: #e6e9f2;
  --muted: #9ca3af;
  --accent: #d4d4d8;
  --accent-2: #a3a3a3;
  --danger: #a1a1aa;
}
html, body { height: 100%; overflow: hidden; }
body {
  margin: 0;
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple SD Gothic Neo", "Noto Sans KR", "Malgun Gothic", sans-serif;
  color: var(--text);
  background: var(--bg);
}
.layout {
  display: grid;
  grid-template-columns: 280px 1fr;
  grid-template-rows: auto 1fr;
  height: 100dvh;
  min-height: 100dvh;
}
header {
  grid-column: 1 / -1;
  position: sticky; top: 0;
  background: var(--panel);
  border-bottom: 1px solid #262626;
  padding: 18px 20px;
  display: grid; place-items: center;
  text-align: center;
}
#current-time { font-size: clamp(50px, 6vw, 100px); letter-spacing: 1.2px; font-variant-numeric: tabular-nums; font-weight: 500; }
.now-label { font-size: 12px; color: var(--muted); letter-spacing: .24em; text-transform: uppercase; margin-bottom: 6px; }

aside {
  background: var(--panel);
  border-right: 1px solid #262626;
  padding: 16px 14px;
  display: grid;
  grid-template-rows: auto 1fr; /* title then scrollable list */
  min-height: 0; /* allow child to shrink and scroll */
  overflow: hidden; /* scrolling happens on the list */
}
aside h2 { margin: 8px 6px 12px; font-size: 16px; color: var(--muted); font-weight: 600; }
.schedule {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 12px;
  grid-auto-rows: minmax(56px, auto);
  min-height: 0;
  overflow: auto; /* scroll only inside the list if needed */
}
.item {
  background: #0f0f0f;
  border: 1px solid #262626;
  border-radius: 10px;
  padding: 14px 16px;
  display: grid;
  grid-template-columns: 1fr auto;
  grid-template-rows: auto;
  gap: 8px 12px;
  align-items: center;
}
.item .time { font-variant-numeric: tabular-nums; color: var(--muted); font-size: 18px; font-weight: 700; text-align: right; white-space: nowrap; }
.item .subject { font-weight: 700; letter-spacing: .3px; font-size: 22px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.item.active { border-color: var(--accent); }
.item.active .subject { color: var(--accent); }
.item.break .subject { color: var(--accent-2); }

main { display: grid; place-items: center; padding: 4vh 3vw; }
.status { text-align: center; display: grid; gap: 16px; }
.status .subject { font-size: clamp(36px, 5vw, 56px); font-weight: 800; }
.status .window { color: var(--muted); font-size: 16px; }
.status .done { color: var(--danger); }

.small { font-size: 12px; color: var(--muted); }
.sr-only { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0; }

@media (max-width: 860px) {
  .layout { grid-template-columns: 1fr; }
  aside { grid-row: 2; border-right: none; border-top: 1px solid #262626; }
  main { grid-row: 3; }
}
</style>
