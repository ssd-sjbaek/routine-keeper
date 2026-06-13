# 스케줄 캘린더 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 교대근무 기록 캘린더를 범용 일정 등록/수정/삭제 캘린더로 교체한다.

**Architecture:** 단일 HTML 파일(`index.html`) 안에서 동작. 기존 shift 관련 CSS/JS/HTML을 제거하고, Firestore `users/{uid}/events` 컬렉션 기반 새 일정 CRUD를 추가한다. 날짜 클릭 시 중앙 모달 팝업으로 일정 목록·등록·수정을 모두 처리한다.

**Tech Stack:** Vanilla JS, Firebase Firestore compat SDK 10.7.1 (CDN), CSS Custom Properties

---

## 파일 구조

| 파일 | 변경 내용 |
|------|-----------|
| `index.html` | CSS 교체, HTML 교체, JS 추가/제거 — 유일한 파일 |

---

### Task 1: 기존 교대근무 코드 제거 + renderMain 정리

교대근무 관련 코드(CSS, JS, HTML)를 전부 제거한다. 이 태스크 완료 후 스케줄 화면은 빈 상태가 된다.

**Files:**
- Modify: `index.html`

---

- [ ] **Step 1: 교대근무 CSS 제거**

`index.html`에서 아래 CSS 블록을 찾아 삭제한다.

삭제 대상 — `.cal-guide` 부터 `.badge-none` 까지 (약 line 359–413):

```css
/* 삭제할 블록 시작 */
.cal-guide {
  text-align: center;
  font-size: 12px;
  color: #99bbcc;
  margin-bottom: 12px;
  position: relative;
  z-index: 1;
}
.calendar {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 5px;
  max-width: 480px;
  margin: 0 auto;
  position: relative;
  z-index: 1;
}
.cal-header {
  text-align: center;
  font-size: 12px;
  color: #99bbcc;
  padding: 4px 0;
  font-weight: 600;
}
.cal-day {
  background: var(--color-card);
  border: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
  border-radius: 10px;
  padding: 6px 2px;
  text-align: center;
  cursor: pointer;
  min-height: 52px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-shadow: 0 1px 4px rgba(100,160,220,0.08);
  transition: background 0.1s;
}
.cal-day:hover { background: color-mix(in srgb, var(--color-bg) 20%, white); }
.day-num { font-size: 13px; font-weight: 600; color: #334455; }
.day-shift { font-size: 10px; margin-top: 3px; font-weight: 700; }
.shift-a  { background: color-mix(in srgb, #c8e0ff 60%, white); }
.shift-a  .day-shift { color: #1565c0; }
.shift-c  { background: color-mix(in srgb, #ffd8c8 60%, white); }
.shift-c  .day-shift { color: #bf360c; }
.shift-do { background: #eeeeee; }
.shift-do .day-shift { color: #666; }

/* 뱃지 */
.badge { border-radius: 8px; padding: 5px 12px; font-size: 12px; font-weight: 600; }
.badge-a  { background: #d4eaff; color: #1565c0; }
.badge-c  { background: #ffe0d4; color: #bf360c; }
.badge-do { background: #e8e8e8; color: #555; }
.badge-none { background: #f5f5f5; color: #aaa; }
/* 삭제할 블록 끝 */
```

- [ ] **Step 2: screen-schedule HTML 교체**

기존 (line 650–657):
```html
  <!-- 스케줄 설정 화면 -->
  <div id="screen-schedule" style="display:none">
    <div class="sub-screen-header">
      <button class="back-btn" onclick="showScreen('main')">← 뒤로</button>
      <span id="schedule-month"></span>
    </div>
    <p class="cal-guide">날짜를 클릭하면 A조 → C조 → DO → 비움 순으로 바뀌어요</p>
    <div id="calendar" class="calendar"></div>
  </div>
```

교체 후:
```html
  <!-- 스케줄 화면 -->
  <div id="screen-schedule" style="display:none">
    <div class="sub-screen-header">
      <button class="back-btn" onclick="showScreen('main')">← 뒤로</button>
      <div class="cal-month-nav">
        <button class="cal-nav-btn" onclick="calNavMonth(-1)">◀</button>
        <span id="schedule-month"></span>
        <button class="cal-nav-btn" onclick="calNavMonth(1)">▶</button>
      </div>
    </div>
    <div id="calendar" class="calendar"></div>
  </div>
```

- [ ] **Step 3: timing-hint div 제거**

`index.html`에서 아래 한 줄을 찾아 삭제한다 (line ~628):
```html
        <div id="timing-hint" class="hint" style="display:none"></div>
```

- [ ] **Step 4: JS — SHIFT_LABELS, SHIFT_CYCLE, getTimingHint 제거**

아래 블록을 찾아 삭제한다 (line ~787–821):
```javascript
    const SHIFT_LABELS = {
      'A':  'A조  09:00–18:00',
      'C':  'C조  14:00–22:00',
      'DO': 'Day Off',
    };

    const SHIFT_CYCLE = ['A', 'C', 'DO', null];
```

그리고 아래 함수도 삭제한다:
```javascript
    // ── 타이밍 힌트 ───────────────────────────────────
    function getTimingHint(shift) {
      if (shift === 'A')  return '✨ 저녁에 하기 좋아요 (퇴근 후 18:00~)';
      if (shift === 'C')  return '✨ 오전에 하기 좋아요 (출근 전 ~14:00)';
      if (shift === 'DO') return '✨ 오늘은 자유롭게 :)';
      return '📅 오늘 스케줄을 먼저 설정해줘요';
    }
```

- [ ] **Step 5: JS — saveShift, loadShift 제거**

아래 함수들을 찾아 삭제한다 (line ~846–860):
```javascript
    async function saveShift(dateKey, shift) {
      const ref = db.collection('users').doc(currentUser.uid)
                    .collection('schedule').doc(dateKey);
      if (shift === null) {
        await ref.delete();
      } else {
        await ref.set({ shift });
      }
    }

    async function loadShift(dateKey) {
      const doc = await db.collection('users').doc(currentUser.uid)
                           .collection('schedule').doc(dateKey).get();
      return doc.exists ? doc.data().shift : null;
    }
```

- [ ] **Step 6: renderMain() 정리**

기존 (line ~1043–1057):
```javascript
    async function renderMain() {
      const today = getTodayKey();
      const [shift, checks, allTodos] = await Promise.all([
        loadShift(today),
        loadRoutineChecks(today),
        loadTodoList(),
      ]);

      const shiftText = shift ? ` · ${SHIFT_LABELS[shift]}` : '';
      document.getElementById('date-hint-bar').textContent = getTodayLabel() + shiftText;

      const timingHintEl = document.getElementById('timing-hint');
      const hintText = getTimingHint(shift);
      timingHintEl.textContent = hintText;
      timingHintEl.style.display = hintText ? '' : 'none';
```

교체 후:
```javascript
    async function renderMain() {
      const today = getTodayKey();
      const [checks, allTodos] = await Promise.all([
        loadRoutineChecks(today),
        loadTodoList(),
      ]);

      document.getElementById('date-hint-bar').textContent = getTodayLabel();
```

- [ ] **Step 7: migrateFromLocalStorage 정리**

기존 (line ~1002–1021) 에서 schedule 관련 부분만 제거한다:
```javascript
    async function migrateFromLocalStorage() {
      const scheduleData = safeParseJSON('schedule');   // ← 삭제
      const routinesData = safeParseJSON('routines');

      const hasSched    = Object.keys(scheduleData).length > 0;  // ← 삭제
      const hasRoutines = Object.keys(routinesData).length > 0;
      if (!hasSched && !hasRoutines) return;             // ← 수정

      const batch   = db.batch();
      const userRef = db.collection('users').doc(currentUser.uid);

      for (const [dateKey, shift] of Object.entries(scheduleData)) {  // ← 삭제
        batch.set(userRef.collection('schedule').doc(dateKey), { shift });  // ← 삭제
      }                                                                // ← 삭제
      for (const [dateKey, checks] of Object.entries(routinesData)) {
        batch.set(userRef.collection('routines').doc(dateKey), checks);
      }

      await batch.commit();
      localStorage.removeItem('schedule');  // ← 삭제
      localStorage.removeItem('routines');
    }
```

수정 후 결과:
```javascript
    async function migrateFromLocalStorage() {
      const routinesData = safeParseJSON('routines');
      if (Object.keys(routinesData).length === 0) return;

      const batch   = db.batch();
      const userRef = db.collection('users').doc(currentUser.uid);
      for (const [dateKey, checks] of Object.entries(routinesData)) {
        batch.set(userRef.collection('routines').doc(dateKey), checks);
      }

      await batch.commit();
      localStorage.removeItem('routines');
    }
```

- [ ] **Step 8: cycleShift, renderCalendar 제거**

아래 함수들을 찾아 삭제한다 (line ~1465–1522):
```javascript
    // ── 달력 화면 ─────────────────────────────────────
    async function renderCalendar() {
      // ... (전체 함수)
    }

    async function cycleShift(dateKey) {
      // ... (전체 함수)
    }
```

- [ ] **Step 9: 브라우저에서 앱 열어 확인**

`index.html`을 직접 브라우저에서 열거나 로컬 서버로 실행한다.
- 스케줄 탭을 누르면 빈 화면이 나오되 에러가 없어야 한다.
- 홈, 통계, 설정 탭이 정상 동작해야 한다.
- 콘솔에 에러가 없어야 한다.

- [ ] **Step 10: 커밋**

```bash
git add index.html
git commit -m "refactor: 교대근무 스케줄 코드 제거 + renderMain 정리"
```

---

### Task 2: 새 캘린더 CSS + 상태 변수

새 캘린더 그리드 스타일과 JS 상태 변수를 추가한다.

**Files:**
- Modify: `index.html`

---

- [ ] **Step 1: 새 캘린더 CSS 추가**

`</style>` 닫는 태그 바로 앞에 아래 CSS를 삽입한다:

```css
    /* ── 새 캘린더 ── */
    .cal-month-nav {
      display: flex;
      align-items: center;
      gap: 10px;
      font-size: 14px;
      font-weight: bold;
      color: var(--color-accent);
    }
    .cal-nav-btn {
      background: none;
      border: none;
      font-size: 16px;
      color: var(--color-accent);
      cursor: pointer;
      padding: 2px 6px;
      border-radius: 6px;
      transition: background 0.1s;
    }
    .cal-nav-btn:hover { background: color-mix(in srgb, var(--color-bg) 40%, white); }

    .calendar {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 4px;
      max-width: 480px;
      margin: 8px auto 0;
      position: relative;
      z-index: 1;
      padding: 0 4px;
    }
    .cal-dow {
      text-align: center;
      font-size: 11px;
      color: #99bbcc;
      padding: 4px 0;
      font-weight: 600;
    }
    .cal-day {
      background: var(--color-card);
      border: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
      border-radius: 8px;
      padding: 4px 3px 5px;
      text-align: center;
      cursor: pointer;
      min-height: 48px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 2px;
      box-shadow: 0 1px 3px rgba(100,160,220,0.07);
      transition: background 0.1s;
      overflow: hidden;
    }
    .cal-day:hover { background: color-mix(in srgb, var(--color-bg) 25%, white); }
    .cal-day.today .day-num {
      background: var(--color-accent);
      color: white;
      border-radius: 50%;
      width: 22px;
      height: 22px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .day-num {
      font-size: 12px;
      font-weight: 600;
      color: #334455;
      line-height: 1;
    }
    .event-chip {
      font-size: 9px;
      background: color-mix(in srgb, var(--color-accent) 15%, white);
      color: var(--color-accent);
      border-radius: 3px;
      padding: 1px 3px;
      width: 100%;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      line-height: 1.3;
    }
    .event-more {
      font-size: 8px;
      color: #99bbcc;
    }

    /* ── 일정 모달 ── */
    .day-modal-overlay {
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.35);
      z-index: 100;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 20px;
    }
    .day-modal {
      background: var(--color-card);
      border: 2px solid var(--color-border);
      border-radius: 16px;
      width: 100%;
      max-width: 360px;
      overflow: hidden;
      box-shadow: 0 8px 32px rgba(0,0,0,0.18);
    }
    .day-modal-header {
      background: linear-gradient(135deg,
        color-mix(in srgb, var(--color-bg) 80%, white),
        color-mix(in srgb, var(--color-bg) 60%, white)
      );
      padding: 12px 14px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      font-size: 13px;
      font-weight: bold;
      color: var(--color-accent);
      border-bottom: 1px solid var(--color-border);
    }
    .day-modal-close {
      background: none;
      border: none;
      font-size: 20px;
      color: #99bbcc;
      cursor: pointer;
      line-height: 1;
      padding: 0 2px;
    }
    #day-event-list { padding: 8px 12px 0; min-height: 40px; }
    .no-events {
      text-align: center;
      color: #aabbcc;
      font-size: 12px;
      padding: 16px 0 8px;
    }
    .ev-item {
      border-bottom: 1px solid color-mix(in srgb, var(--color-border) 30%, white);
      padding: 6px 0;
    }
    .ev-item:last-child { border-bottom: none; }
    .ev-item-main {
      display: flex;
      align-items: center;
      gap: 8px;
      cursor: pointer;
      font-size: 13px;
      color: #334455;
    }
    .ev-time-badge {
      background: color-mix(in srgb, var(--color-accent) 12%, white);
      color: var(--color-accent);
      border-radius: 6px;
      padding: 2px 6px;
      font-size: 10px;
      flex-shrink: 0;
    }
    .ev-item-detail {
      padding: 6px 0 4px 32px;
      font-size: 12px;
      color: #667788;
    }
    .ev-memo-text { margin-bottom: 6px; white-space: pre-wrap; }
    .ev-item-actions { display: flex; gap: 8px; }
    .ev-item-actions button {
      background: none;
      border: 1px solid var(--color-border);
      border-radius: 6px;
      padding: 3px 10px;
      font-size: 11px;
      color: var(--color-accent);
      cursor: pointer;
      font-family: inherit;
    }
    .day-add-btn {
      display: block;
      width: calc(100% - 24px);
      margin: 10px 12px 12px;
      padding: 9px;
      background: var(--color-accent);
      color: white;
      border: none;
      border-radius: 10px;
      font-size: 13px;
      cursor: pointer;
      font-family: inherit;
    }
    .ev-form-inner { padding: 12px 14px 14px; display: flex; flex-direction: column; gap: 8px; }
    .ev-input {
      width: 100%;
      padding: 8px 10px;
      border: 1.5px solid var(--color-border);
      border-radius: 8px;
      font-size: 13px;
      font-family: inherit;
      background: color-mix(in srgb, var(--color-card) 90%, var(--color-bg));
      color: #334455;
      outline: none;
    }
    .ev-input:focus { border-color: var(--color-accent); }
    .ev-memo { resize: vertical; min-height: 64px; }
    .ev-form-btns { display: flex; gap: 8px; justify-content: flex-end; margin-top: 4px; }
    .ev-cancel-btn {
      padding: 7px 16px;
      border: 1.5px solid var(--color-border);
      border-radius: 8px;
      background: none;
      color: #667788;
      font-size: 13px;
      cursor: pointer;
      font-family: inherit;
    }
    .ev-save-btn {
      padding: 7px 16px;
      border: none;
      border-radius: 8px;
      background: var(--color-accent);
      color: white;
      font-size: 13px;
      cursor: pointer;
      font-family: inherit;
    }
    .ev-save-btn:disabled { background: #ccd8e4; cursor: not-allowed; }
```

- [ ] **Step 2: 상태 변수 추가**

`index.html`의 기존 상태 변수 블록 끝 (line ~785 근처, `let currentTab = 'today';` 아래)에 추가한다:

```javascript
    let calViewYear  = new Date().getFullYear();
    let calViewMonth = new Date().getMonth() + 1;
    let currentModalDate = null;
    let editingEventId   = null;
```

- [ ] **Step 3: escapeHtml 유틸 함수 추가**

`getTodayKey()` 함수 바로 위에 추가한다:

```javascript
    function escapeHtml(str) {
      return String(str)
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#39;');
    }
```

- [ ] **Step 4: 브라우저에서 확인**

앱을 열고 스케줄 탭 클릭.
- 달력 그리드 자리는 비어있지만 에러 없어야 함
- `calNavMonth is not defined` 에러 없어야 함 (아직 renderCalendar 없어서 빈 캘린더)

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: 새 캘린더 CSS + 상태 변수 추가"
```

---

### Task 3: Firestore 이벤트 함수 + 새 renderCalendar()

일정 CRUD Firestore 함수와 새 renderCalendar()를 구현한다.

**Files:**
- Modify: `index.html`

---

- [ ] **Step 1: Firestore 이벤트 함수 추가**

기존 `saveShift`/`loadShift`를 삭제한 자리 (line ~846 근처, `saveRoutineCheck` 바로 위)에 추가한다:

```javascript
    // ── 이벤트 Firestore 함수 ─────────────────────────
    async function loadEventsForMonth(year, month) {
      const pad = n => String(n).padStart(2, '0');
      const start = `${year}-${pad(month)}-01`;
      const end   = `${year}-${pad(month)}-31`;
      const snap = await db.collection('users').doc(currentUser.uid)
        .collection('events')
        .where('date', '>=', start)
        .where('date', '<=', end)
        .get();
      return snap.docs.map(d => ({ id: d.id, ...d.data() }));
    }

    async function loadEventsForDay(dateKey) {
      const snap = await db.collection('users').doc(currentUser.uid)
        .collection('events')
        .where('date', '==', dateKey)
        .get();
      const events = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      events.sort((a, b) => (a.time || '99:99').localeCompare(b.time || '99:99'));
      return events;
    }

    async function loadEventById(eventId) {
      const doc = await db.collection('users').doc(currentUser.uid)
        .collection('events').doc(eventId).get();
      return doc.exists ? { id: doc.id, ...doc.data() } : null;
    }

    async function saveEvent(data) {
      await db.collection('users').doc(currentUser.uid)
        .collection('events').add({
          ...data,
          createdAt: firebase.firestore.FieldValue.serverTimestamp(),
        });
    }

    async function updateEvent(eventId, data) {
      await db.collection('users').doc(currentUser.uid)
        .collection('events').doc(eventId).set(data, { merge: true });
    }

    async function deleteEvent(eventId) {
      await db.collection('users').doc(currentUser.uid)
        .collection('events').doc(eventId).delete();
    }
```

- [ ] **Step 2: 새 renderCalendar() + calNavMonth() 추가**

Task 1에서 삭제한 `renderCalendar` 자리 (line ~1464 근처, `// ── 달력 화면` 주석이 있던 곳)에 추가한다:

```javascript
    // ── 달력 화면 ─────────────────────────────────────
    async function renderCalendar() {
      const year  = calViewYear;
      const month = calViewMonth;

      document.getElementById('schedule-month').textContent = `${year}년 ${month}월`;

      const events = await loadEventsForMonth(year, month);
      const byDate = {};
      events.forEach(ev => {
        if (!byDate[ev.date]) byDate[ev.date] = [];
        byDate[ev.date].push(ev);
      });

      const firstWeekday = new Date(year, month - 1, 1).getDay();
      const daysInMonth  = new Date(year, month, 0).getDate();
      const todayKey     = getTodayKey();

      const cal = document.getElementById('calendar');
      cal.innerHTML = '';

      ['일','월','화','수','목','금','토'].forEach(w => {
        const h = document.createElement('div');
        h.className = 'cal-dow';
        h.textContent = w;
        cal.appendChild(h);
      });

      for (let i = 0; i < firstWeekday; i++) {
        cal.appendChild(document.createElement('div'));
      }

      for (let day = 1; day <= daysInMonth; day++) {
        const m   = String(month).padStart(2, '0');
        const d   = String(day).padStart(2, '0');
        const dateKey  = `${year}-${m}-${d}`;
        const dayEvs   = byDate[dateKey] || [];

        const cell = document.createElement('div');
        cell.className = 'cal-day' + (dateKey === todayKey ? ' today' : '');

        const numSpan = document.createElement('span');
        numSpan.className = 'day-num';
        numSpan.textContent = day;
        cell.appendChild(numSpan);

        if (dayEvs.length > 0) {
          const chip = document.createElement('span');
          chip.className = 'event-chip';
          chip.textContent = dayEvs[0].title;
          cell.appendChild(chip);
          if (dayEvs.length > 1) {
            const more = document.createElement('span');
            more.className = 'event-more';
            more.textContent = `+${dayEvs.length - 1}`;
            cell.appendChild(more);
          }
        }

        cell.onclick = () => openDayModal(dateKey);
        cal.appendChild(cell);
      }
    }

    function calNavMonth(delta) {
      calViewMonth += delta;
      if (calViewMonth > 12) { calViewYear++;  calViewMonth = 1; }
      if (calViewMonth < 1)  { calViewYear--;  calViewMonth = 12; }
      renderCalendar();
    }
```

- [ ] **Step 3: 브라우저에서 확인**

앱을 열고 스케줄 탭 클릭.
- 이번 달 캘린더가 정상 렌더링되어야 한다 (일정 없으면 숫자만)
- ◀ ▶ 버튼으로 월 이동이 되어야 한다
- 오늘 날짜에 `--color-accent` 원형 배경이 보여야 한다
- 날짜 클릭 시 콘솔에 `openDayModal is not defined` 에러가 나도 괜찮음 (Task 4에서 구현)

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 이벤트 Firestore 함수 + 새 renderCalendar 구현"
```

---

### Task 4: 일정 모달 HTML + JS 구현

날짜 클릭 시 나타나는 중앙 모달(일정 목록 + 등록/수정 폼)을 완성한다.

**Files:**
- Modify: `index.html`

---

- [ ] **Step 1: 모달 HTML 추가**

`screen-settings` 닫는 `</div>` 태그 바로 뒤, `<script>` 태그 바로 앞에 삽입한다:

```html
  <!-- 일정 모달 -->
  <div id="day-modal-overlay" class="day-modal-overlay" style="display:none" onclick="onModalOverlayClick(event)">
    <div class="day-modal">
      <div class="day-modal-header">
        <span id="day-modal-title"></span>
        <button class="day-modal-close" onclick="closeDayModal()">×</button>
      </div>
      <div id="day-modal-list">
        <div id="day-event-list"></div>
        <button class="day-add-btn" onclick="showEventForm(currentModalDate)">+ 일정 추가</button>
      </div>
      <div id="day-modal-form" style="display:none">
        <div class="ev-form-inner">
          <input id="ev-title" class="ev-input" type="text" placeholder="일정 제목 *" maxlength="50" oninput="updateSaveBtn()">
          <input id="ev-time" class="ev-input" type="time">
          <textarea id="ev-memo" class="ev-input ev-memo" placeholder="메모 (선택)" maxlength="200"></textarea>
          <div class="ev-form-btns">
            <button class="ev-cancel-btn" onclick="showEventList()">취소</button>
            <button id="ev-save-btn" class="ev-save-btn" onclick="saveEventForm()" disabled>저장</button>
          </div>
        </div>
      </div>
    </div>
  </div>
```

- [ ] **Step 2: 모달 JS 추가**

`calNavMonth` 함수 바로 뒤에 추가한다:

```javascript
    // ── 일정 모달 ─────────────────────────────────────
    async function openDayModal(dateKey) {
      currentModalDate = dateKey;
      editingEventId   = null;

      const [y, m, d]  = dateKey.split('-').map(Number);
      const weekdays   = ['일','월','화','수','목','금','토'];
      const dt         = new Date(y, m - 1, d);
      document.getElementById('day-modal-title').textContent =
        `${y}년 ${m}월 ${d}일 (${weekdays[dt.getDay()]})`;

      document.getElementById('day-modal-overlay').style.display = '';
      showEventList();
      await renderDayEventList(dateKey);
    }

    function closeDayModal() {
      document.getElementById('day-modal-overlay').style.display = 'none';
      currentModalDate = null;
      editingEventId   = null;
    }

    function onModalOverlayClick(e) {
      if (e.target === document.getElementById('day-modal-overlay')) closeDayModal();
    }

    async function renderDayEventList(dateKey) {
      const events = await loadEventsForDay(dateKey);
      const list   = document.getElementById('day-event-list');
      list.innerHTML = '';

      if (events.length === 0) {
        list.innerHTML = '<p class="no-events">등록된 일정이 없어요</p>';
        return;
      }

      events.forEach(ev => {
        const item = document.createElement('div');
        item.className = 'ev-item';
        const timeStr = ev.time || '종일';
        item.innerHTML = `
          <div class="ev-item-main" onclick="toggleEvDetail('${ev.id}')">
            <span class="ev-time-badge">${escapeHtml(timeStr)}</span>
            <span>${escapeHtml(ev.title)}</span>
          </div>
          <div class="ev-item-detail" id="ev-detail-${ev.id}" style="display:none">
            <p class="ev-memo-text">${ev.memo ? escapeHtml(ev.memo) : ''}</p>
            <div class="ev-item-actions">
              <button onclick="showEventForm('${escapeHtml(ev.date)}','${ev.id}')">수정</button>
              <button onclick="confirmDeleteEvent('${ev.id}')">삭제</button>
            </div>
          </div>
        `;
        list.appendChild(item);
      });
    }

    function toggleEvDetail(evId) {
      const el = document.getElementById('ev-detail-' + evId);
      if (el) el.style.display = el.style.display === 'none' ? '' : 'none';
    }

    function showEventList() {
      document.getElementById('day-modal-list').style.display = '';
      document.getElementById('day-modal-form').style.display = 'none';
    }

    async function showEventForm(dateKey, eventId) {
      editingEventId = eventId || null;
      document.getElementById('day-modal-list').style.display = 'none';
      document.getElementById('day-modal-form').style.display = '';

      document.getElementById('ev-title').value = '';
      document.getElementById('ev-time').value  = '';
      document.getElementById('ev-memo').value  = '';

      if (eventId) {
        const ev = await loadEventById(eventId);
        if (ev) {
          document.getElementById('ev-title').value = ev.title;
          document.getElementById('ev-time').value  = ev.time  || '';
          document.getElementById('ev-memo').value  = ev.memo  || '';
        }
      }
      updateSaveBtn();
    }

    function updateSaveBtn() {
      const title = document.getElementById('ev-title').value.trim();
      document.getElementById('ev-save-btn').disabled = !title;
    }

    async function saveEventForm() {
      const title = document.getElementById('ev-title').value.trim();
      if (!title) return;
      const time = document.getElementById('ev-time').value  || null;
      const memo = document.getElementById('ev-memo').value.trim() || null;

      if (editingEventId) {
        await updateEvent(editingEventId, { title, time, memo });
      } else {
        await saveEvent({ date: currentModalDate, title, time, memo });
      }

      showEventList();
      await renderDayEventList(currentModalDate);
      await renderCalendar();
    }

    async function confirmDeleteEvent(eventId) {
      if (!confirm('이 일정을 삭제할까요?')) return;
      await deleteEvent(eventId);
      await renderDayEventList(currentModalDate);
      await renderCalendar();
    }
```

- [ ] **Step 3: 브라우저에서 전체 기능 확인**

앱을 열고 순서대로 테스트한다:

1. 스케줄 탭 클릭 → 이번 달 캘린더 렌더링 확인
2. 아무 날짜 클릭 → 중앙 모달 팝업, "등록된 일정이 없어요" 표시 확인
3. "+ 일정 추가" 클릭 → 폼으로 전환 확인
4. 제목 비워두면 저장 버튼 비활성 확인
5. 제목 입력 → 저장 버튼 활성화 확인
6. 제목 + 시간 + 메모 입력 후 저장 → 목록에 나타남 확인
7. 같은 날 일정 2개 등록 → 캘린더 셀에 첫 번째 제목 + "+1" 표시 확인
8. 일정 탭 → 수정/삭제 버튼 나타남 확인
9. 수정 클릭 → 폼에 기존 값 채워짐 확인
10. 삭제 클릭 → confirm 후 삭제 확인
11. 모달 배경 클릭 → 모달 닫힘 확인
12. ◀ ▶ 버튼으로 월 이동 확인
13. 콘솔에 에러 없음 확인

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 일정 등록/수정/삭제 모달 구현"
```

---

## 자체 검토

### 스펙 커버리지

| 스펙 항목 | 구현 태스크 |
|-----------|------------|
| 월 캘린더 그리드, 이전/다음 달 이동 | Task 2 (CSS), Task 3 (JS) |
| 일정 있는 날짜에 제목 칩 + "+N" | Task 3 `renderCalendar()` |
| 오늘 날짜 강조 | Task 2 CSS `.today` |
| 날짜 클릭 → 중앙 모달 | Task 4 `openDayModal` |
| 모달: 날짜 헤더 + × 닫기 | Task 4 HTML + JS |
| 일정 목록 (시간 뱃지 + 제목) | Task 4 `renderDayEventList` |
| 일정 탭 → 수정/삭제 버튼 | Task 4 `toggleEvDetail` |
| "+ 일정 추가" 버튼 → 폼 전환 | Task 4 `showEventForm` |
| 폼: 제목(필수), 시간, 메모 | Task 4 HTML + `saveEventForm` |
| 저장 버튼 제목 없으면 비활성 | Task 4 `updateSaveBtn` |
| Firestore `users/{uid}/events` | Task 3 Firestore 함수 |
| 기존 shift 코드 전부 제거 | Task 1 |
| CSS 변수 테마 유지 | Task 2 CSS (var() 사용) |

모든 항목 커버됨.
