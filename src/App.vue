<template>
  <div class="layout">
    <header>
      <div id="current-time" aria-live="polite">{{ currentTimeText }}</div>
      <button class="fs-btn" type="button" @click="toggleFullscreen" aria-label="전체화면">
        ⛶
      </button>
      <!-- <div class="tester">
        <label>
          <input type="checkbox" v-model="testMode" />
          테스트 모드
        </label>
        <input type="time" v-model="testTime" step="60" />
      </div> -->
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
        <div id="current-subject" class="subject" :class="{ done: statusState === 'done' }">
          {{ currentSubjectText }}
        </div>
        <div id="window-time" class="window">{{ windowText }}</div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

const testMode = ref(false)
const testTime = ref('')

// === 설정: 오늘 진행할 모의고사 시간표 ===
const SCHEDULE = [
  { subject: '국어', start: '08:40', end: '10:00', sit: '08:15' },
  { subject: '수학', start: '10:30', end: '12:10', sit: '10:20' },
  { subject: '점심', start: '12:10', end: '13:00'},
  { subject: '영어', start: '13:10', end: '14:20', sit: '13:00' },
  { subject: '한국사', start: '14:50', end: '15:20', sit: '14:40' },
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

function fmtHM(d) {
  const h = String(d.getHours()).padStart(2, '0')
  const m = String(d.getMinutes()).padStart(2, '0')
  return `${h}:${m}`
}

const effectiveNow = computed(() => {
  if (testMode.value && testTime.value) return todayAt(testTime.value)
  return now.value
})

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
const currentTimeText = computed(() => clockFmt.format(effectiveNow.value))

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

const status = computed(() => computeStatus(effectiveNow.value))
const statusState = computed(() => status.value.state)

const currentSubjectText = computed(() => {
  const st = status.value
  if (st.state === 'in') return st.window.break ? '휴식' : `${st.window.subject}`
  if (st.state === 'pre' || st.state === 'gap') return `다음: ${st.next.subject}`
  return '오늘 일정 종료'
})

const windowText = computed(() => {
  const st = status.value
  if (st.state === 'in') {
    let base = `${st.window.start} – ${st.window.end}`
    if (st.window.subject === '점심') {
      base += `\n13:00까지 입실 완료\n(12:30 점심)`
    }
    return base
  }
  if (st.state === 'pre' || st.state === 'gap') {
    const base = `${st.next.start} – ${st.next.end}`
    if (st.next && st.next.sit) return `${base}\n${st.next.sit}까지 입실 완료`
    return base
  }
  return '모든 과목 완료'
})

function isActive(w) {
  const st = status.value
  return st.state === 'in' && st.window.subject === w.subject
}

function toggleFullscreen() {
  const el = document.documentElement
  if (!document.fullscreenElement) {
    (el.requestFullscreen && el.requestFullscreen())
  } else {
    (document.exitFullscreen && document.exitFullscreen())
  }
}

let intervalId
onMounted(() => {
  // 테스트 모드 기본값: 현재 시각(HH:MM)
  testTime.value = fmtHM(new Date())
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
  grid-template-columns: 360px 1fr;
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
#current-time { font-size: clamp(64px, 7vw, 120px); letter-spacing: 1.2px; font-variant-numeric: tabular-nums; font-weight: 500; }
.now-label { font-size: 12px; color: var(--muted); letter-spacing: .24em; text-transform: uppercase; margin-bottom: 6px; }

.tester { display: flex; gap: 10px; align-items: center; margin-top: 8px; font-size: 12px; color: var(--muted); }
.tester input[type="time"] { background: #0f0f0f; color: var(--text); border: 1px solid #262626; padding: 4px 6px; border-radius: 6px; }
.tester input[type="checkbox"] { accent-color: #a3a3a3; }

aside {
  background: var(--panel);
  border-right: 1px solid #262626;
  padding: 16px 14px;
  display: grid;
  grid-template-rows: auto 1fr;
  min-height: 0; 
  overflow: hidden;
}
aside h2 { margin: 8px 6px 12px; font-size: 22px; color: var(--muted); font-weight: 600; }
.schedule {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 1fr;
  gap: 4px;
  grid-auto-rows: minmax(64px, auto);
  min-height: 0;
  overflow: auto;
  scrollbar-width: none;  /* Firefox */
  -ms-overflow-style: none;  /* Internet Explorer and Edge */ 
}
.schedule li {
  height: 40px;
}
.item {
  background: #0f0f0f;
  border: 1px solid #262626;
  border-radius: 10px;
  padding: 18px 20px;
  display: grid;
  grid-template-columns: 1fr auto;
  grid-template-rows: auto;
  gap: 8px 12px;
  align-items: center;
}
.item .time { font-variant-numeric: tabular-nums; color: var(--muted); font-size: 22px; font-weight: 700; text-align: right; white-space: nowrap; }
.item .subject { font-weight: 700; letter-spacing: .3px; font-size: 28px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.item.active { border-color: var(--accent); }
.item.active .subject { color: var(--accent); }
.item.break .subject { color: var(--accent-2); }

main { display: grid; place-items: center; padding: 4vh 3vw; }
.status { text-align: center; display: grid; gap: 16px; }
.status .subject { font-size: clamp(64px, 8vw, 120px); font-weight: 800; }
.status .window { color: var(--muted); font-size: 80px; white-space: pre-line; color: white; }
.status .done { color: var(--danger); }

.small { font-size: 12px; color: var(--muted); }
.sr-only { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0; }

@media (max-width: 860px) {
  .layout { grid-template-columns: 1fr; }
  aside { grid-row: 2; border-right: none; border-top: 1px solid #262626; }
  main { grid-row: 3; }
}

header { position: relative; }
.fs-btn { position: absolute; right: 16px; top: 12px; background: #108fcf; color: var(--text); border: 1px solid #262626; border-radius: 8px; padding: 8px 12px; font-size: 16px; cursor: pointer; }
.fs-btn:hover { filter: brightness(1.1); }
</style>
