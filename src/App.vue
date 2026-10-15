<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type Worker = { id: string; name: string; role: string; initials: string; color: string; target: number }
type Shift = { id: string; date: string; start: string; end: string; workerId: string; kind: 'Dyżur' | 'Zajęcia'; title?: string }
type Request = { id: string; workerId: string; date: string; type: string; note: string; status: 'Do decyzji' | 'Zaakceptowana' | 'Odrzucona' }

const workers = ref<Worker[]>(load('chwyt-workers', [
  { id: 'marta', name: 'Marta Nowak', role: 'Etat · 160 h', initials: 'MN', color: 'lilac', target: 160 },
  { id: 'jakub', name: 'Jakub Zieliński', role: 'Etat · 160 h', initials: 'JZ', color: 'mint', target: 160 },
  { id: 'ola', name: 'Ola Wiśniewska', role: 'Etat · 160 h', initials: 'OW', color: 'peach', target: 160 },
  { id: 'kuba', name: 'Kuba Maj', role: 'Student', initials: 'KM', color: 'blue', target: 36 },
  { id: 'ania', name: 'Ania Lis', role: 'Student', initials: 'AL', color: 'yellow', target: 28 },
]))

const today = new Date()
const viewedMonth = ref(new Date(2026, 9, 1))
const selectedDate = ref('2026-10-15')
const viewMode = ref<'miesiąc' | 'zespół'>('miesiąc')
const showModal = ref(false)
const modalType = ref<'shift' | 'request' | 'employee'>('shift')
const toast = ref('')
const published = ref(false)

const initialShifts: Shift[] = []
for (let day = 1; day <= 31; day++) {
  const date = `2026-10-${String(day).padStart(2, '0')}`
  const dow = new Date(2026, 9, day).getDay()
  const weekend = dow === 0 || dow === 6
  const people = weekend ? ['marta', 'jakub', 'ola', 'kuba', 'ania'] : ['marta', 'jakub', 'ola']
  const offset = (day + 1) % people.length
  const ordered = [...people.slice(offset), ...people.slice(0, offset)]
  const windows = weekend ? [['09:00', '15:00'], ['15:00', '21:00']] : [['08:00', '15:00'], ['15:00', '22:00']]
  windows.forEach((w, i) => initialShifts.push({ id: `${date}-${i}`, date, start: w[0], end: w[1], workerId: ordered[i], kind: 'Dyżur' }))
}
const initialRequests: Request[] = [
  { id: 'r1', workerId: 'marta', date: '2026-10-24', type: 'Prośba o wolne', note: 'Ważna sprawa rodzinna', status: 'Do decyzji' },
  { id: 'r2', workerId: 'kuba', date: '2026-10-31', type: 'Dostępność', note: 'Mogę pracować cały dzień', status: 'Do decyzji' },
  { id: 'r3', workerId: 'ola', date: '2026-10-18', type: 'Preferowana zmiana', note: 'Najchętniej poranna', status: 'Zaakceptowana' },
]

function load<T>(key: string, fallback: T): T {
  try { const value = localStorage.getItem(key); return value ? JSON.parse(value) as T : fallback } catch { return fallback }
}
const shifts = ref<Shift[]>(load('chwyt-shifts', initialShifts))
const requests = ref<Request[]>(load('chwyt-requests', initialRequests))
watch(shifts, value => localStorage.setItem('chwyt-shifts', JSON.stringify(value)), { deep: true })
watch(requests, value => localStorage.setItem('chwyt-requests', JSON.stringify(value)), { deep: true })
watch(workers, value => localStorage.setItem('chwyt-workers', JSON.stringify(value)), { deep: true })

const monthTitle = computed(() => new Intl.DateTimeFormat('pl-PL', { month: 'long', year: 'numeric' }).format(viewedMonth.value))
const days = computed(() => {
  const year = viewedMonth.value.getFullYear(), month = viewedMonth.value.getMonth()
  const first = new Date(year, month, 1), count = new Date(year, month + 1, 0).getDate()
  const offset = (first.getDay() + 6) % 7
  return Array.from({ length: Math.ceil((offset + count) / 7) * 7 }, (_, i) => {
    const date = new Date(year, month, i - offset + 1)
    return { date: dateKey(date), day: date.getDate(), inMonth: date.getMonth() === month, weekend: date.getDay() === 0 || date.getDay() === 6 }
  })
})
const monthDays = computed(() => days.value.filter(d => d.inMonth))
const monthShift = computed(() => shifts.value.filter(s => s.date.startsWith(monthKey.value)))
const monthKey = computed(() => `${viewedMonth.value.getFullYear()}-${String(viewedMonth.value.getMonth() + 1).padStart(2, '0')}`)
const monthHours = computed(() => monthShift.value.reduce((sum, s) => sum + duration(s.start, s.end), 0))
const openRequests = computed(() => requests.value.filter(r => r.status === 'Do decyzji'))
const hoursByWorker = computed(() => Object.fromEntries(workers.value.map(w => [w.id, monthShift.value.filter(s => s.workerId === w.id).reduce((n, s) => n + duration(s.start, s.end), 0)])) as Record<string, number>)
const openDays = computed(() => monthDays.value.filter(day => !holidays.value[day.date]).length)
const coverageDays = computed(() => monthDays.value.filter(day => !holidays.value[day.date] && (() => {
  const weekend = day.weekend, start = weekend ? 9 * 60 : 8 * 60, end = weekend ? 21 * 60 : 22 * 60
  const assigned = shiftsFor(day.date).filter(s => s.kind === 'Dyżur').sort((a, b) => a.start.localeCompare(b.start))
  let cursor = start
  for (const shift of assigned) {
    const shiftStart = clockMinutes(shift.start), shiftEnd = clockMinutes(shift.end)
    if (shiftStart > cursor) return false
    cursor = Math.max(cursor, shiftEnd)
    if (cursor >= end) return true
  }
  return false
})()).length)
const holidays = computed(() => holidayMap(viewedMonth.value.getFullYear()))
const pendingShift = ref({ date: '2026-10-15', start: '08:00', end: '15:00', workerId: 'marta', kind: 'Dyżur' as 'Dyżur' | 'Zajęcia', title: '' })
const pendingRequest = ref({ workerId: 'marta', date: '2026-10-24', type: 'Prośba o wolne', note: '' })
const pendingEmployee = ref({ name: '', contract: 'Etat', target: 160 })
const selectedWorker = ref('all')

function dateKey(date: Date) { return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}` }
function duration(start: string, end: string) { const [a, b] = [start, end].map(t => { const [h, m] = t.split(':').map(Number); return h * 60 + m }); return Math.max(0, (b - a) / 60) }
function clockMinutes(time: string) { const [h, m] = time.split(':').map(Number); return h * 60 + m }
function holidayMap(year: number) {
  const fixed: Record<string, string> = { '01-01': 'Nowy Rok', '01-06': 'Trzech Króli', '05-01': 'Święto Pracy', '05-03': 'Święto Konstytucji 3 Maja', '08-15': 'Wniebowzięcie NMP', '11-01': 'Wszystkich Świętych', '11-11': 'Święto Niepodległości', '12-24': 'Wigilia', '12-25': 'Boże Narodzenie', '12-26': 'Drugi dzień Świąt' }
  const a = year % 19, b = Math.floor(year / 100), c = year % 100, d = Math.floor(b / 4), e = b % 4, f = Math.floor((b + 8) / 25), g = Math.floor((b - f + 1) / 3), h = (19 * a + b - d - g + 15) % 30, i = Math.floor(c / 4), k = c % 4, l = (32 + 2 * e + 2 * i - h - k) % 7, m = Math.floor((a + 11 * h + 22 * l) / 451)
  const easter = new Date(year, Math.floor((h + l - 7 * m + 114) / 31) - 1, ((h + l - 7 * m + 114) % 31) + 1)
  const result: Record<string, string> = {}
  for (const [md, name] of Object.entries(fixed)) result[`${year}-${md}`] = name
  for (const [offset, name] of [[0, 'Wielkanoc'], [1, 'Poniedziałek Wielkanocny'], [49, 'Zielone Świątki'], [60, 'Boże Ciało']] as [number, string][]) { const date = new Date(easter); date.setDate(date.getDate() + offset); result[dateKey(date)] = name }
  return result
}
function worker(id: string) { return workers.value.find(w => w.id === id) ?? workers.value[0] }
function shiftsFor(date: string) { return shifts.value.filter(s => s.date === date).sort((a, b) => a.start.localeCompare(b.start)) }
function shortDate(date: string) { return new Intl.DateTimeFormat('pl-PL', { day: 'numeric', month: 'long' }).format(new Date(`${date}T12:00:00`)) }
function changeMonth(amount: number) { viewedMonth.value = new Date(viewedMonth.value.getFullYear(), viewedMonth.value.getMonth() + amount, 1); selectedDate.value = dateKey(new Date(viewedMonth.value.getFullYear(), viewedMonth.value.getMonth(), 1)) }
function openShift(date: string) { selectedDate.value = date; pendingShift.value = { date, start: '08:00', end: '15:00', workerId: 'marta', kind: 'Dyżur', title: '' }; modalType.value = 'shift'; showModal.value = true }
function openRequest() { modalType.value = 'request'; showModal.value = true }
function openEmployee() { pendingEmployee.value = { name: '', contract: 'Etat', target: 160 }; modalType.value = 'employee'; showModal.value = true }
function saveShift() {
  const { date, start, end, workerId, kind, title } = pendingShift.value
  if (duration(start, end) < 4) { notify('Zmiana musi trwać co najmniej 4 godziny.'); return }
  if (start >= end) { notify('Godzina zakończenia musi być późniejsza.'); return }
  shifts.value.push({ id: crypto.randomUUID(), date, start, end, workerId, kind, title: title || undefined }); showModal.value = false; notify(kind === 'Dyżur' ? 'Dodano dyżur do grafiku.' : 'Dodano zajęcia do grafiku.')
}
function saveRequest() {
  if (!pendingRequest.value.note.trim()) { notify('Dodaj krótką treść prośby.'); return }
  requests.value.unshift({ ...pendingRequest.value, id: crypto.randomUUID(), status: 'Do decyzji' }); showModal.value = false; notify('Prośba została dodana.')
}
function saveEmployee() {
  const name = pendingEmployee.value.name.trim()
  if (!name) { notify('Wpisz imię i nazwisko pracownika.'); return }
  const initials = name.split(/\s+/).slice(0, 2).map(part => part[0]).join('').toUpperCase()
  const colors = ['lilac', 'mint', 'peach', 'blue', 'yellow']
  workers.value.push({ id: crypto.randomUUID(), name, role: pendingEmployee.value.contract === 'Etat' ? `Etat · ${pendingEmployee.value.target} h` : 'Student', initials, color: colors[workers.value.length % colors.length], target: pendingEmployee.value.contract === 'Etat' ? Number(pendingEmployee.value.target) : 32 })
  showModal.value = false; notify('Dodano pracownika do zespołu.')
}
function resolveRequest(id: string, status: 'Zaakceptowana' | 'Odrzucona') { const request = requests.value.find(r => r.id === id); if (request) request.status = status; notify(status === 'Zaakceptowana' ? 'Prośba zaakceptowana.' : 'Prośba odrzucona.') }
function generateSchedule() {
  const output: Shift[] = []
  let fullTimeIndex = 0, studentIndex = 0
  const fullTimers = workers.value.filter(w => w.target === 160), students = workers.value.filter(w => w.target !== 160)
  for (const day of monthDays.value) {
    if (holidays.value[day.date]) continue
    const dow = new Date(`${day.date}T12:00:00`).getDay(), weekend = dow === 0 || dow === 6
    const windows = weekend ? [['09:00', '15:00'], ['15:00', '21:00']] : [['08:00', '15:00'], ['15:00', '22:00']]
    windows.forEach((window, index) => {
      let person: Worker
      if (weekend && index === 1 && students.length && studentIndex % 3 !== 2) person = students[studentIndex++ % students.length]
      else person = fullTimers[fullTimeIndex++ % fullTimers.length]
      output.push({ id: `${day.date}-${index}-${crypto.randomUUID()}`, date: day.date, start: window[0], end: window[1], workerId: person.id, kind: 'Dyżur' })
    })
  }
  shifts.value = [...shifts.value.filter(s => !s.date.startsWith(monthKey.value) || s.kind === 'Zajęcia'), ...output]
  published.value = false
  notify('Ułożono propozycję obsady. Sprawdź godziny i zajęcia przed publikacją.')
}
function notify(message: string) { toast.value = message; window.setTimeout(() => { toast.value = '' }, 3200) }
function publish() { published.value = true; notify('Grafik oznaczony jako opublikowany.') }
function filteredShifts(date: string) { return shiftsFor(date).filter(s => selectedWorker.value === 'all' || s.workerId === selectedWorker.value) }
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <div class="brand"><div class="brand-mark"><svg viewBox="0 0 32 32" fill="none"><path d="M7 25V7h5.8l7.4 10.1V7H26v18h-5.6L13 14.7V25H7Z" fill="currentColor"/><circle cx="24.5" cy="6.5" r="3.5" fill="#C8F36B"/></svg></div><span>chwyt<span class="brand-dot">.</span></span></div>
      <div class="club-switch"><div class="club-icon">W</div><div><b>Ściana Północ</b><small>Warszawa · 1 lokal</small></div><span class="chevron">⌄</span></div>
      <div class="nav-label">PRZESTRZEŃ</div>
      <button class="nav-item" :class="{ active: viewMode === 'miesiąc' }" @click="viewMode = 'miesiąc'"><span class="nav-icon">▦</span> Grafik <span class="nav-count">{{ openRequests.length }}</span></button>
      <button class="nav-item" :class="{ active: viewMode === 'zespół' }" @click="viewMode = 'zespół'"><span class="nav-icon">♙</span> Zespół</button>
      <button class="nav-item" @click="openRequest"><span class="nav-icon">♡</span> Prośby <span v-if="openRequests.length" class="request-pip"></span></button>
      <div class="nav-label second-label">USTAWIENIA</div>
      <button class="nav-item" @click="notify('Ustawienia świąt i godzin lokalu pojawią się w kolejnym etapie.')"><span class="nav-icon">⚙</span> Godziny i święta</button>
      <div class="sidebar-bottom"><div class="sidebar-tip"><span class="tip-spark">✳</span><b>Mała wskazówka</b><p>Najpierw dodaj zajęcia, a potem ułóż dyżury. Łatwiej utrzymać ciągłość obsady.</p><button @click="viewMode = 'miesiąc'">Zobacz grafik <span>→</span></button></div><div class="account"><div class="avatar avatar-dark">MH</div><div><b>Michał Hadamik</b><small>Administrator</small></div><span class="more">···</span></div></div>
    </aside>

    <main class="main-area">
      <header class="topbar"><div class="breadcrumb">Ściana Północ <span>/</span> <b>{{ viewMode === 'miesiąc' ? 'Grafik' : 'Zespół' }}</b></div><div class="top-actions"><span class="saved"><i></i> Zapisano</span><button class="icon-button" aria-label="Powiadomienia" @click="notify('Nie masz nowych powiadomień.')">♧<i class="notify-dot"></i></button><div class="avatar avatar-dark top-avatar">MH</div></div></header>
      <div class="content">
        <section class="page-heading"><div><div class="eyebrow"><span class="eyebrow-line"></span> PLANOWANIE ZESPOŁU</div><h1>{{ viewMode === 'miesiąc' ? 'Grafik pracy' : 'Twój zespół' }}<span class="title-period" v-if="viewMode === 'miesiąc'">·</span></h1><p>{{ viewMode === 'miesiąc' ? 'Zaplanuj dyżury, zajęcia i czas dla całego zespołu.' : 'Osoby, które dbają o dobrą asekurację na ścianie.' }}</p></div><div class="heading-actions"><button class="button button-quiet" @click="openRequest"><span class="plus">＋</span> Dodaj prośbę</button><button class="button button-primary" @click="generateSchedule"><svg viewBox="0 0 20 20" fill="none"><path d="M10 2.5v3M10 14.5v3M2.5 10h3m9 0h3M4.7 4.7l2.1 2.1m6.4 6.4 2.1 2.1m0-10.6-2.1 2.1m-6.4 6.4-2.1 2.1" stroke="currentColor" stroke-width="1.6" stroke-linecap="round"/></svg> Ułóż grafik</button></div></section>
        <div class="prototype-notice"><span>i</span><p><b>Wersja demonstracyjna.</b> Dane zapisują się tylko w tej przeglądarce. Nie wpisuj prawdziwych danych pracowników — logowanie i wspólna baza nie są jeszcze podłączone.</p></div>

        <section v-if="viewMode === 'miesiąc'" class="stats-grid">
          <article class="stat-card"><div class="stat-label">GODZINY W GRAFIKU <span class="stat-icon mint-icon">◷</span></div><div class="stat-value">{{ monthHours }}<small> h</small></div><div class="stat-foot"><span class="stat-dot green-dot"></span> {{ monthShift.length }} zaplanowanych zmian</div><div class="stat-graph"><i v-for="n in 21" :key="n" :style="{height: `${12 + ((n * 17) % 31)}px`}"></i></div></article>
          <article class="stat-card"><div class="stat-label">OBSADA DYŻURÓW <span class="stat-icon blue-icon">⌖</span></div><div class="stat-value">{{ coverageDays }}<small> / {{ openDays }} dni</small></div><div class="progress-track"><span :style="{width: `${openDays ? coverageDays / openDays * 100 : 0}%`}"></span></div><div class="stat-foot">Pełna obsada godzin otwarcia · święta wolne</div></article>
          <article class="stat-card"><div class="stat-label">PROŚBY DO ROZPATRZENIA <span class="stat-icon peach-icon">♡</span></div><div class="stat-value">{{ String(openRequests.length).padStart(2, '0') }}</div><div class="stat-foot"><span class="request-tag">{{ openRequests.length ? 'Wymagają uwagi' : 'Wszystko rozpatrzone' }}</span> {{ openRequests.length ? '· warto sprawdzić' : '' }}</div><div class="avatar-stack"><div v-for="r in openRequests.slice(0, 3)" :key="r.id" class="avatar tiny-avatar" :class="`avatar-${worker(r.workerId).color}`">{{ worker(r.workerId).initials }}</div></div></article>
        </section>

        <section v-if="viewMode === 'miesiąc'" class="workspace-grid">
          <div class="calendar-card">
            <div class="calendar-toolbar"><div><div class="month-title">{{ monthTitle }}</div><div class="month-subtitle">{{ monthDays.length }} dni · <span class="status-dot"></span> {{ published ? 'Opublikowany' : 'Wersja robocza' }}</div></div><div class="calendar-actions"><div class="filter-select"><span class="filter-person">♙</span><select v-model="selectedWorker" aria-label="Filtruj po pracowniku"><option value="all">Cały zespół</option><option v-for="person in workers" :key="person.id" :value="person.id">{{ person.name }}</option></select><span>⌄</span></div><button class="month-nav" aria-label="Poprzedni miesiąc" @click="changeMonth(-1)">‹</button><button class="month-nav" aria-label="Następny miesiąc" @click="changeMonth(1)">›</button><button class="today-button" @click="viewedMonth = new Date(today.getFullYear(), today.getMonth(), 1)">Dziś</button></div></div>
            <div class="calendar-grid"><div v-for="day in ['PON', 'WT', 'ŚR', 'CZW', 'PT', 'SOB', 'NIEDZ']" :key="day" class="weekday">{{ day }}</div>
              <div v-for="day in days" :key="day.date" class="day-cell" :class="{ 'outside-month': !day.inMonth, 'weekend-cell': day.weekend, 'holiday-cell': holidays[day.date], 'selected-cell': day.date === selectedDate }" @click="selectedDate = day.date">
                <div class="day-number" :class="{ 'today-number': day.date === dateKey(new Date()) }"><span>{{ day.day }}</span><span v-if="day.date === dateKey(new Date())" class="today-label">dziś</span></div><div v-if="holidays[day.date]" class="holiday-label" :title="holidays[day.date]">{{ holidays[day.date] }}</div>
                <div class="day-events"><button v-for="shift in filteredShifts(day.date).slice(0, 2)" :key="shift.id" class="event-chip" :class="[`event-${worker(shift.workerId).color}`, { 'class-event': shift.kind === 'Zajęcia' }]" @click.stop="selectedDate = day.date"><span class="event-time">{{ shift.start }}</span><span class="event-name">{{ shift.kind === 'Zajęcia' ? (shift.title || 'Zajęcia') : worker(shift.workerId).name.split(' ')[0] }}</span></button><button v-if="filteredShifts(day.date).length > 2" class="more-events" @click.stop="selectedDate = day.date">+ {{ filteredShifts(day.date).length - 2 }} więcej</button><button class="add-event" :aria-label="`Dodaj do ${day.day} ${monthTitle}`" @click.stop="openShift(day.date)">＋</button></div>
              </div>
            </div>
            <div class="calendar-legend"><span class="legend-label">ZESPÓŁ</span><span v-for="person in workers" :key="person.id" class="legend-person"><i :class="`legend-${person.color}`"></i>{{ person.name.split(' ')[0] }}</span><span class="legend-note"><i></i> Zajęcia</span><button class="export-link" @click="notify('Synchronizacja z Google Calendar będzie dostępna po podłączeniu konta.')">↗ Google Calendar</button></div>
          </div>

          <aside class="right-rail">
            <section class="rail-card requests-card"><div class="rail-heading"><div><div class="rail-title">Prośby zespołu <span v-if="openRequests.length" class="count-pill">{{ openRequests.length }}</span></div><p>Wnioski czekające na decyzję</p></div><button class="small-more" aria-label="Więcej prośb" @click="openRequest">···</button></div>
              <div v-if="openRequests.length" class="request-list"><article v-for="request in openRequests" :key="request.id" class="request-item"><div class="request-top"><div class="avatar request-avatar" :class="`avatar-${worker(request.workerId).color}`">{{ worker(request.workerId).initials }}</div><div class="request-person"><b>{{ worker(request.workerId).name }}</b><small>{{ request.type }} · {{ shortDate(request.date) }}</small></div><span class="pending-dot"></span></div><p class="request-note">„{{ request.note }}”</p><div class="request-controls"><button class="approve-button" @click="resolveRequest(request.id, 'Zaakceptowana')">✓ Akceptuj</button><button class="decline-button" @click="resolveRequest(request.id, 'Odrzucona')">Odrzuć</button></div></article></div><div v-else class="empty-state"><span>✳</span><b>Brak nowych próśb</b><small>Kiedy ktoś doda prośbę, pojawi się tutaj.</small></div>
              <button class="all-requests" @click="openRequest">Dodaj prośbę <span>↗</span></button>
            </section>
            <section class="rail-card team-card"><div class="rail-heading"><div><div class="rail-title">Zespół <span class="muted-count">{{ workers.length }}</span></div><p>Godziny w tym miesiącu</p></div><button class="small-more" aria-label="Pokaż zespół" @click="viewMode = 'zespół'">↗</button></div><div class="team-list"><div v-for="person in workers" :key="person.id" class="team-row"><div class="avatar team-avatar" :class="`avatar-${person.color}`">{{ person.initials }}</div><div class="team-info"><b>{{ person.name }}</b><small>{{ person.role }}</small></div><div class="hours-number">{{ hoursByWorker[person.id] }}<small> / {{ person.target }} h</small><div class="hours-track"><i :class="`bar-${person.color}`" :style="{width: `${Math.min(100, hoursByWorker[person.id] / person.target * 100)}%`}"></i></div></div></div></div><button class="all-requests team-link" @click="viewMode = 'zespół'">Zobacz zespół <span>→</span></button></section>
            <section class="tip-card"><span class="tip-spark">✳</span><div><b>Nie zostawiaj pustych dni</b><p>Ułóż propozycję grafiku i sprawdź każdy dyżur przed publikacją.</p><button @click="generateSchedule">Ułóż grafik <span>→</span></button></div><span class="tip-decoration">✳</span></section>
          </aside>
        </section>

        <section v-else class="people-page"><div class="people-intro"><div><h2>Godziny i dostępność</h2><p>Podsumowanie zespołu za {{ monthTitle }}.</p></div><button class="button button-primary" @click="openEmployee"><span class="plus">＋</span> Dodaj pracownika</button></div><div class="people-grid"><article v-for="person in workers" :key="person.id" class="person-card"><div class="person-card-top"><div class="avatar person-avatar" :class="`avatar-${person.color}`">{{ person.initials }}</div><span class="person-status"><i></i> Aktywny</span></div><h3>{{ person.name }}</h3><p>{{ person.role }}</p><div class="person-hours"><span>Godziny zaplanowane</span><b>{{ hoursByWorker[person.id] }} <small>/ {{ person.target }} h</small></b><div class="hours-track"><i :class="`bar-${person.color}`" :style="{width: `${Math.min(100, hoursByWorker[person.id] / person.target * 100)}%`}"></i></div></div><button class="person-action" @click="selectedWorker = person.id; viewMode = 'miesiąc'">Zobacz zmiany <span>→</span></button></article></div></section>

        <footer class="page-footer"><span>CHWYT <i>·</i> GRAFIK DLA DOBREJ ASEKURACJI</span><span>Prototyp · dane zapisują się na tym urządzeniu</span></footer>
      </div>
    </main>

    <div v-if="showModal" class="modal-backdrop" @click.self="showModal = false"><div class="modal"><div class="modal-head"><div><div class="eyebrow"><span class="eyebrow-line"></span>{{ modalType === 'shift' ? 'PLANOWANIE' : 'ZESPÓŁ' }}</div><h2>{{ modalType === 'shift' ? 'Dodaj do grafiku' : modalType === 'request' ? 'Nowa prośba' : 'Dodaj pracownika' }}</h2></div><button class="modal-close" @click="showModal = false">×</button></div>
      <form v-if="modalType === 'shift'" @submit.prevent="saveShift"><label>Rodzaj wpisu<select v-model="pendingShift.kind"><option>Dyżur</option><option>Zajęcia</option></select></label><label>Pracownik<select v-model="pendingShift.workerId"><option v-for="person in workers" :key="person.id" :value="person.id">{{ person.name }}</option></select></label><label>Data<input v-model="pendingShift.date" type="date" required /></label><div class="form-row"><label>Od<input v-model="pendingShift.start" type="time" required /></label><label>Do<input v-model="pendingShift.end" type="time" required /></label></div><label v-if="pendingShift.kind === 'Zajęcia'">Nazwa zajęć<input v-model="pendingShift.title" placeholder="np. Grupa początkująca" /></label><p class="form-hint">Minimalna długość dyżuru: 4 godziny. Zajęcia należy uwzględnić przy sprawdzaniu obsady.</p><div class="modal-actions"><button type="button" class="button button-quiet" @click="showModal = false">Anuluj</button><button type="submit" class="button button-primary">Dodaj wpis</button></div></form>
      <form v-else-if="modalType === 'request'" @submit.prevent="saveRequest"><label>Pracownik<select v-model="pendingRequest.workerId"><option v-for="person in workers" :key="person.id" :value="person.id">{{ person.name }}</option></select></label><label>Rodzaj prośby<select v-model="pendingRequest.type"><option>Prośba o wolne</option><option>Dostępność</option><option>Preferowana zmiana</option><option>Zamiana zmiany</option></select></label><label>Data<input v-model="pendingRequest.date" type="date" required /></label><label>Treść<textarea v-model="pendingRequest.note" rows="3" placeholder="Napisz, czego dotyczy prośba…" required></textarea></label><div class="modal-actions"><button type="button" class="button button-quiet" @click="showModal = false">Anuluj</button><button type="submit" class="button button-primary">Wyślij prośbę</button></div></form>
      <form v-else @submit.prevent="saveEmployee"><label>Imię i nazwisko<input v-model="pendingEmployee.name" placeholder="np. Aleksandra Kowalska" required /></label><label>Typ zatrudnienia<select v-model="pendingEmployee.contract"><option>Etat</option><option>Student</option></select></label><label v-if="pendingEmployee.contract === 'Etat'">Miesięczny cel godzin<input v-model.number="pendingEmployee.target" type="number" min="1" max="240" required /></label><div class="modal-actions"><button type="button" class="button button-quiet" @click="showModal = false">Anuluj</button><button type="submit" class="button button-primary">Dodaj do zespołu</button></div></form>
    </div></div>
    <Transition name="toast"><div v-if="toast" class="toast-message"><span>✓</span>{{ toast }}</div></Transition>
  </div>
</template>
