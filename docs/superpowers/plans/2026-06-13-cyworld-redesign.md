# 싸이월드 미니홈피 리디자인 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 루틴 지킴이 앱 전체 UI를 싸이월드 미니홈피 스타일로 교체하고, 사용자가 설정에서 색상 4개를 각각 자유롭게 변경할 수 있게 한다.

**Architecture:** 단일 `index.html` 파일. CSS 커스텀 프로퍼티(변수) 4개로 전체 색상 관리. 색상 설정은 localStorage에 저장되며 앱 로드 시 `applyTheme()`로 복원. 기존 Firebase 로직/데이터 구조는 변경 없음.

**Tech Stack:** Vanilla JS, CSS Custom Properties, localStorage, 기존 Firebase compat SDK

---

## 파일 구조

- **Modify:** `index.html` (유일한 파일 — 모든 변경은 여기)

---

### Task 1: CSS 변수 도입 + 보케 배경

**Files:**
- Modify: `index.html` — `<style>` 최상단 + `<body>` 태그 바로 아래 + `<script>` 내부

- [ ] **Step 1: `:root` CSS 변수를 `<style>` 최상단에 추가**

`index.html`의 `<style>` 태그 안, `* { box-sizing: ... }` 바로 앞에 삽입:

```css
:root {
  --color-bg:     #b2e0f7;
  --color-border: #88ccee;
  --color-accent: #4488bb;
  --color-card:   #ffffff;
}
```

- [ ] **Step 2: `body` 스타일을 CSS 변수 기반으로 교체**

기존:
```css
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f0f7f4;
  color: #2d3a35;
  min-height: 100vh;
  padding: 20px;
}
```

교체:
```css
body {
  font-family: 'Malgun Gothic', '맑은 고딕', -apple-system, sans-serif;
  background: #f0f8ff;
  color: #334455;
  min-height: 100vh;
  padding: 20px;
  position: relative;
  overflow-x: hidden;
}
```

- [ ] **Step 3: `.bokeh-bg` CSS를 `body` 스타일 바로 뒤에 추가**

```css
.bokeh-bg {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
}
.bokeh-bg span {
  position: absolute;
  border-radius: 50%;
  filter: blur(50px);
  opacity: 0.55;
}
.bokeh-bg .b1 { width: 240px; height: 240px; background: var(--color-bg); top: -60px; left: -70px; }
.bokeh-bg .b2 { width: 180px; height: 180px; background: var(--color-bg); top: 100px; right: -50px; opacity: 0.4; }
.bokeh-bg .b3 { width: 280px; height: 280px; background: var(--color-bg); bottom: 40px; left: -80px; opacity: 0.35; }
.bokeh-bg .b4 { width: 160px; height: 160px; background: var(--color-bg); bottom: -40px; right: 10px; opacity: 0.45; }
.bokeh-bg .b5 { width: 140px; height: 140px; background: var(--color-bg); top: 45%; left: 35%; opacity: 0.3; }
```

- [ ] **Step 4: `<body>` 태그 바로 뒤에 보케 HTML 삽입**

`index.html`의 `<body>` 여는 태그 바로 다음 줄에:

```html
<!-- 보케 배경 -->
<div class="bokeh-bg">
  <span class="b1"></span>
  <span class="b2"></span>
  <span class="b3"></span>
  <span class="b4"></span>
  <span class="b5"></span>
</div>
```

- [ ] **Step 5: `applyTheme()` + `saveTheme()` 함수를 JS 상수 선언부 (`const DEFAULT_ROUTINES`) 바로 앞에 추가**

```javascript
// ── 테마 (색상 커스터마이징) ──────────────────────────
const DEFAULT_THEME = {
  bg:     '#b2e0f7',
  border: '#88ccee',
  accent: '#4488bb',
  card:   '#ffffff',
};

function applyTheme(theme) {
  const t = theme || DEFAULT_THEME;
  const root = document.documentElement.style;
  root.setProperty('--color-bg',     t.bg);
  root.setProperty('--color-border', t.border);
  root.setProperty('--color-accent', t.accent);
  root.setProperty('--color-card',   t.card);
}

function saveTheme(theme) {
  localStorage.setItem('cy_theme', JSON.stringify(theme));
  applyTheme(theme);
}

function loadTheme() {
  try {
    const stored = localStorage.getItem('cy_theme');
    return stored ? JSON.parse(stored) : DEFAULT_THEME;
  } catch {
    return DEFAULT_THEME;
  }
}
```

- [ ] **Step 6: 앱 맨 처음에 `applyTheme(loadTheme())` 호출 추가**

`firebase.initializeApp(firebaseConfig);` 바로 다음 줄에:

```javascript
applyTheme(loadTheme());
```

- [ ] **Step 7: 브라우저에서 확인**

브라우저로 `index.html`을 열어 보케 배경 원들이 보이는지 확인.

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "feat: CSS 변수 + 보케 배경 + applyTheme 추가"
```

---

### Task 2: 메인 화면 레이아웃 재구성

**Files:**
- Modify: `index.html` — HTML 구조 + renderMain() + showScreen()

기존 `screen-main`의 `.header` 제거, `.mh-header` 추가, `.bottom-nav` 추가. `.shift-row` 제거, 날짜/시프트 정보를 `#date-hint-bar`로 통합.

- [ ] **Step 1: `screen-main` HTML 전체 교체**

기존:
```html
<!-- 메인 화면 -->
<div id="screen-main" style="display:none">
  <div class="header">
    <span id="today-date">날짜 로딩 중...</span>
    <div class="header-actions">
      <button onclick="showScreen('schedule')">📅 스케줄 설정</button>
      <button onclick="handleSignOut()">로그아웃</button>
    </div>
  </div>
  <div class="card">
    <div class="tab-bar">
      <button class="tab-btn active" id="tab-today" onclick="switchTab('today')">오늘</button>
      <button class="tab-btn" id="tab-stats" onclick="switchTab('stats')">📊 통계</button>
    </div>
    <div id="panel-today">
      <div class="shift-row">
        <h1>오늘</h1>
        <span id="shift-badge" class="badge badge-none">미설정</span>
      </div>
      <div id="timing-hint" class="hint">오늘 스케줄을 먼저 설정해줘요</div>
      <div id="routine-section" class="routine-list"></div>
      <div id="todo-section" class="todo-section"></div>
    </div>
    <div id="panel-stats" style="display:none"></div>
  </div>
</div>
```

교체:
```html
<!-- 메인 화면 -->
<div id="screen-main" style="display:none">
  <div class="card">
    <!-- 싸이월드 헤더 -->
    <div class="mh-header">
      <div class="mh-avatar">🌸</div>
      <div class="mh-info">
        <div class="mh-title">루틴 지킴이</div>
        <div class="mh-sub" id="user-display-name">로딩 중...</div>
      </div>
      <div class="mh-streak" id="streak-header-badge" style="display:none"></div>
    </div>
    <!-- 탭바 -->
    <div class="tab-bar">
      <button class="tab-btn active" id="tab-today" onclick="switchTab('today')">오늘</button>
      <button class="tab-btn" id="tab-stats" onclick="switchTab('stats')">📊 통계</button>
    </div>
    <!-- 오늘 패널 -->
    <div id="panel-today">
      <div class="date-hint-bar" id="date-hint-bar">로딩 중...</div>
      <div id="timing-hint" class="hint" style="display:none"></div>
      <div id="routine-section"></div>
      <div id="todo-section" class="todo-section"></div>
    </div>
    <!-- 통계 패널 -->
    <div id="panel-stats" style="display:none"></div>
    <!-- 하단 네비 -->
    <div class="bottom-nav">
      <div class="nav-item" id="nav-home" onclick="showScreen('main')">
        <span class="nav-icon">🏠</span><span>홈</span>
      </div>
      <div class="nav-item" id="nav-schedule" onclick="showScreen('schedule')">
        <span class="nav-icon">📅</span><span>스케줄</span>
      </div>
      <div class="nav-item" id="nav-settings" onclick="showScreen('settings')">
        <span class="nav-icon">⚙️</span><span>설정</span>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 2: `screen-schedule` HTML 교체 (헤더 업데이트)**

기존:
```html
<!-- 스케줄 설정 화면 -->
<div id="screen-schedule" style="display:none">
  <div class="header">
    <button onclick="showScreen('main')">← 뒤로</button>
    <span id="schedule-month"></span>
  </div>
  <p class="cal-guide">날짜를 클릭하면 A조 → C조 → DO → 비움 순으로 바뀌어요</p>
  <div id="calendar" class="calendar"></div>
</div>
```

교체:
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

- [ ] **Step 3: `screen-settings` HTML을 `screen-schedule` 바로 아래에 추가**

```html
<!-- 설정 화면 -->
<div id="screen-settings" style="display:none">
  <div class="sub-screen-header">
    <button class="back-btn" onclick="showScreen('main')">← 뒤로</button>
    <span>⚙️ 설정</span>
  </div>
  <div class="settings-card">
    <div class="settings-section-title">🎨 색상 꾸미기</div>
    <div id="color-pickers"></div>
    <button class="save-theme-btn" onclick="onSaveTheme()">저장하기 ✓</button>
  </div>
  <div class="settings-card" style="margin-top:12px">
    <div class="settings-section-title">계정</div>
    <button class="logout-btn" onclick="handleSignOut()">로그아웃</button>
  </div>
</div>
```

- [ ] **Step 4: `renderMain()` 내에서 날짜/시프트 업데이트 코드 수정**

`renderMain()` 함수 내부에서 아래 코드를:
```javascript
document.getElementById('today-date').textContent = getTodayLabel();

const badge = document.getElementById('shift-badge');
if (shift) {
  badge.textContent = SHIFT_LABELS[shift];
  badge.className = `badge badge-${shift.toLowerCase()}`;
} else {
  badge.textContent = '미설정';
  badge.className = 'badge badge-none';
}
document.getElementById('timing-hint').textContent = getTimingHint(shift);
```

아래로 교체:
```javascript
const shiftText = shift ? ` · ${SHIFT_LABELS[shift]}` : '';
document.getElementById('date-hint-bar').textContent = getTodayLabel() + shiftText;

const timingHintEl = document.getElementById('timing-hint');
const hint = getTimingHint(shift);
timingHintEl.textContent = hint;
timingHintEl.style.display = '';

// 사용자 이름 표시
if (currentUser && currentUser.displayName) {
  document.getElementById('user-display-name').textContent = currentUser.displayName + '의 홈';
}
```

- [ ] **Step 5: `showScreen()` 함수에 `settings` 처리 추가**

기존 `showScreen` 함수:
```javascript
function showScreen(name) {
  document.getElementById('screen-login').style.display    = name === 'login'    ? '' : 'none';
  document.getElementById('screen-main').style.display     = name === 'main'     ? '' : 'none';
  document.getElementById('screen-schedule').style.display = name === 'schedule' ? '' : 'none';
  if (name === 'schedule') renderCalendar();
  if (name === 'main') {
    if (currentTab !== 'today') switchTab('today');
    renderMain();
  }
}
```

교체:
```javascript
function showScreen(name) {
  document.getElementById('screen-login').style.display    = name === 'login'    ? '' : 'none';
  document.getElementById('screen-main').style.display     = name === 'main'     ? '' : 'none';
  document.getElementById('screen-schedule').style.display = name === 'schedule' ? '' : 'none';
  document.getElementById('screen-settings').style.display = name === 'settings' ? '' : 'none';
  // 하단 네비 활성 상태
  ['home','schedule','settings'].forEach(id => {
    const el = document.getElementById('nav-' + id);
    if (el) el.classList.remove('active');
  });
  if (name === 'main')     { const el = document.getElementById('nav-home');     if (el) el.classList.add('active'); }
  if (name === 'schedule') { const el = document.getElementById('nav-schedule'); if (el) el.classList.add('active'); }
  if (name === 'settings') { const el = document.getElementById('nav-settings'); if (el) el.classList.add('active'); }
  if (name === 'schedule') renderCalendar();
  if (name === 'settings') renderSettings();
  if (name === 'main') {
    if (currentTab !== 'today') switchTab('today');
    renderMain();
  }
}
```

- [ ] **Step 6: 빈 `renderSettings()` 함수 추가 (Task 4에서 완성)**

`showScreen` 바로 아래에:

```javascript
function renderSettings() {
  // Task 4에서 완성
}
```

- [ ] **Step 7: 브라우저에서 확인**

앱을 열어 새 헤더, 날짜 힌트바, 하단 네비가 렌더링되는지 확인. 스케줄/설정 화면 이동 동작 확인.

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "feat: 메인 화면 레이아웃 재구성 - mh-header, bottom-nav, date-hint-bar"
```

---

### Task 3: 전체 CSS 싸이월드 스타일 적용

**Files:**
- Modify: `index.html` — `<style>` 섹션 전체 개편

기존 초록 계열 색상(`#2e7d5e`, `#f0f7f4`, `#88a89c` 등)을 CSS 변수 기반으로 교체하고, 새 컴포넌트 스타일 추가.

- [ ] **Step 1: `body` 이후 전체 CSS를 아래 내용으로 교체**

`<style>` 태그 전체를 아래로 교체한다. (`:root` 블록은 Task 1에서 이미 추가했으므로 포함):

```css
:root {
  --color-bg:     #b2e0f7;
  --color-border: #88ccee;
  --color-accent: #4488bb;
  --color-card:   #ffffff;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: 'Malgun Gothic', '맑은 고딕', -apple-system, sans-serif;
  background: #f0f8ff;
  color: #334455;
  min-height: 100vh;
  padding: 20px;
  position: relative;
  overflow-x: hidden;
}

/* 보케 배경 */
.bokeh-bg { position: fixed; inset: 0; pointer-events: none; z-index: 0; }
.bokeh-bg span { position: absolute; border-radius: 50%; filter: blur(50px); opacity: 0.55; }
.bokeh-bg .b1 { width: 240px; height: 240px; background: var(--color-bg); top: -60px; left: -70px; }
.bokeh-bg .b2 { width: 180px; height: 180px; background: var(--color-bg); top: 100px; right: -50px; opacity: 0.4; }
.bokeh-bg .b3 { width: 280px; height: 280px; background: var(--color-bg); bottom: 40px; left: -80px; opacity: 0.35; }
.bokeh-bg .b4 { width: 160px; height: 160px; background: var(--color-bg); bottom: -40px; right: 10px; opacity: 0.45; }
.bokeh-bg .b5 { width: 140px; height: 140px; background: var(--color-bg); top: 45%; left: 35%; opacity: 0.3; }

/* 로그인 화면 */
#screen-login {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: calc(100vh - 40px);
  position: relative;
  z-index: 1;
}
.login-card {
  background: var(--color-card);
  border: 2.5px solid var(--color-border);
  border-radius: 20px;
  padding: 40px 32px;
  box-shadow: 0 4px 24px rgba(100,160,220,0.18);
  max-width: 360px;
  width: 100%;
  text-align: center;
}
.login-card h1 {
  font-size: 22px;
  font-weight: 700;
  color: var(--color-accent);
  margin-bottom: 6px;
}
.login-card p {
  font-size: 13px;
  color: #99bbcc;
  margin-bottom: 28px;
  line-height: 1.6;
}
.btn-google {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  width: 100%;
  padding: 12px 20px;
  background: white;
  border: 1.5px solid var(--color-border);
  border-radius: 12px;
  font-size: 14px;
  font-weight: 500;
  color: var(--color-accent);
  cursor: pointer;
  transition: background 0.15s;
  font-family: inherit;
}
.btn-google:hover { background: #f4fbff; }

/* 메인 카드 */
.card {
  background: var(--color-card);
  border: 2.5px solid var(--color-border);
  border-radius: 20px;
  box-shadow: 0 4px 24px rgba(100,160,220,0.15);
  max-width: 480px;
  margin: 0 auto;
  position: relative;
  z-index: 1;
  overflow: hidden;
}

/* 싸이월드 헤더 */
.mh-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px 12px;
  background: color-mix(in srgb, var(--color-bg) 30%, white);
  border-bottom: 2px solid var(--color-border);
}
.mh-avatar {
  width: 42px; height: 42px;
  background: color-mix(in srgb, var(--color-bg) 60%, white);
  border-radius: 50%;
  border: 2px solid var(--color-border);
  display: flex; align-items: center; justify-content: center;
  font-size: 20px; flex-shrink: 0;
}
.mh-info { flex: 1; min-width: 0; }
.mh-title { font-size: 15px; font-weight: bold; color: var(--color-accent); }
.mh-sub { font-size: 11px; color: color-mix(in srgb, var(--color-accent) 60%, #aaa); margin-top: 2px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.mh-streak {
  background: color-mix(in srgb, var(--color-bg) 40%, white);
  border: 1px solid var(--color-border);
  border-radius: 20px;
  padding: 4px 10px;
  font-size: 11px;
  color: var(--color-accent);
  white-space: nowrap;
  flex-shrink: 0;
}

/* 탭바 */
.tab-bar {
  display: flex;
  border-bottom: 2px solid var(--color-border);
  background: color-mix(in srgb, var(--color-bg) 10%, white);
}
.tab-btn {
  flex: 1;
  background: none;
  border: none;
  border-bottom: 2.5px solid transparent;
  margin-bottom: -2px;
  padding: 10px 8px;
  font-size: 13px;
  color: #aabbcc;
  cursor: pointer;
  font-family: inherit;
  transition: color 0.15s;
}
.tab-btn.active {
  color: var(--color-accent);
  border-bottom-color: var(--color-accent);
  font-weight: 700;
  background: white;
}

/* 날짜 힌트바 */
.date-hint-bar {
  padding: 8px 16px;
  font-size: 11px;
  color: color-mix(in srgb, var(--color-accent) 60%, #aaa);
  text-align: center;
  background: color-mix(in srgb, var(--color-bg) 8%, white);
  border-bottom: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
}

/* 타이밍 힌트 */
.hint {
  font-size: 12px;
  color: var(--color-accent);
  background: color-mix(in srgb, var(--color-bg) 15%, white);
  border-radius: 0;
  padding: 8px 16px;
  border-bottom: 1px solid color-mix(in srgb, var(--color-border) 30%, white);
}

/* 섹션 공통 */
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  padding: 12px 14px 0;
}
.section-title {
  font-size: 12px;
  font-weight: 700;
  color: var(--color-accent);
}
.section-btn {
  background: color-mix(in srgb, var(--color-bg) 20%, white);
  border: 1px solid var(--color-border);
  border-radius: 10px;
  padding: 3px 10px;
  font-size: 11px;
  color: var(--color-accent);
  cursor: pointer;
  font-family: inherit;
}
.section-btn:hover { background: color-mix(in srgb, var(--color-bg) 40%, white); }

/* 루틴 목록 */
.routine-list { padding: 0 12px; }
.routine-item {
  display: flex;
  align-items: center;
  gap: 10px;
  background: color-mix(in srgb, var(--color-card) 100%, transparent);
  border: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
  border-radius: 10px;
  padding: 10px 12px;
  cursor: pointer;
  font-size: 14px;
  margin-bottom: 5px;
  transition: background 0.15s;
  user-select: none;
}
.routine-item:hover { background: color-mix(in srgb, var(--color-bg) 15%, white); }
.routine-item input[type="checkbox"] {
  width: 16px; height: 16px;
  accent-color: var(--color-accent);
  cursor: pointer;
}
.routine-item.done { background: color-mix(in srgb, var(--color-bg) 12%, white); }
.routine-item.done .routine-name { text-decoration: line-through; color: #aabbcc; }
.progress {
  text-align: right;
  font-size: 12px;
  color: #aabbcc;
  padding: 6px 14px 10px;
}

/* 드래그 핸들 */
.drag-handle {
  cursor: grab;
  padding: 0 8px 0 2px;
  color: #ccc;
  font-size: 16px;
  user-select: none;
  flex-shrink: 0;
}
.drag-handle:active { cursor: grabbing; }

/* 삭제 버튼 */
.routine-delete, .todo-delete {
  margin-left: auto;
  background: none;
  border: none;
  color: #ccc;
  font-size: 16px;
  cursor: pointer;
  padding: 0 4px;
  line-height: 1;
}
.routine-delete:hover, .todo-delete:hover { color: #e57373; }

/* 입력 행 */
.add-input-row {
  display: flex;
  gap: 8px;
  margin: 8px 12px 10px;
}
.add-input {
  flex: 1;
  border: 1.5px solid var(--color-border);
  border-radius: 10px;
  padding: 8px 12px;
  font-size: 13px;
  outline: none;
  font-family: inherit;
  color: #334455;
}
.add-input:focus { border-color: var(--color-accent); }
.add-btn {
  background: var(--color-accent);
  color: white;
  border: none;
  border-radius: 10px;
  padding: 8px 14px;
  font-size: 13px;
  cursor: pointer;
  font-family: inherit;
}
.add-btn:hover { opacity: 0.85; }

/* 할 일 섹션 */
.todo-section {
  margin-top: 0;
  padding-top: 0;
  border-top: 1px solid color-mix(in srgb, var(--color-border) 30%, white);
  margin: 8px 0 0;
}
.todo-item {
  display: flex;
  align-items: center;
  gap: 10px;
  background: color-mix(in srgb, var(--color-card) 100%, transparent);
  border: 1px solid color-mix(in srgb, var(--color-border) 30%, white);
  border-radius: 10px;
  padding: 10px 12px;
  font-size: 14px;
  margin-bottom: 5px;
  margin: 0 12px 5px;
}
.todo-item.done .todo-text { text-decoration: line-through; color: #aabbcc; }
.todo-item input[type="checkbox"] {
  width: 16px; height: 16px;
  accent-color: var(--color-accent);
  cursor: pointer;
  flex-shrink: 0;
}
.todo-text { flex: 1; }

/* 하단 네비 */
.bottom-nav {
  display: flex;
  justify-content: space-around;
  padding: 10px 0 12px;
  border-top: 2px solid var(--color-border);
  background: color-mix(in srgb, var(--color-bg) 10%, white);
}
.nav-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 3px;
  font-size: 10px;
  color: #aabbcc;
  cursor: pointer;
  padding: 2px 16px;
  border-radius: 10px;
  transition: color 0.15s;
}
.nav-item.active { color: var(--color-accent); font-weight: bold; }
.nav-item:hover { color: var(--color-accent); }
.nav-icon { font-size: 18px; }

/* 서브 화면 헤더 (스케줄, 설정) */
.sub-screen-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  max-width: 480px;
  margin: 0 auto 16px;
  font-size: 14px;
  color: #99bbcc;
  position: relative;
  z-index: 1;
}
.back-btn {
  background: color-mix(in srgb, var(--color-bg) 30%, white);
  border: 1px solid var(--color-border);
  border-radius: 10px;
  padding: 6px 14px;
  color: var(--color-accent);
  cursor: pointer;
  font-size: 13px;
  font-family: inherit;
}
.back-btn:hover { background: color-mix(in srgb, var(--color-bg) 50%, white); }

/* 달력 */
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

/* 뱃지 (달력 화면용) */
.badge { border-radius: 8px; padding: 5px 12px; font-size: 12px; font-weight: 600; }
.badge-a  { background: #d4eaff; color: #1565c0; }
.badge-c  { background: #ffe0d4; color: #bf360c; }
.badge-do { background: #e8e8e8; color: #555; }
.badge-none { background: #f5f5f5; color: #aaa; }

/* 설정 화면 */
#screen-schedule, #screen-settings { position: relative; z-index: 1; }
.settings-card {
  background: var(--color-card);
  border: 2px solid var(--color-border);
  border-radius: 16px;
  padding: 16px;
  max-width: 480px;
  margin: 0 auto;
  box-shadow: 0 3px 16px rgba(100,160,220,0.12);
}
.settings-section-title {
  font-size: 12px;
  font-weight: bold;
  color: var(--color-accent);
  margin-bottom: 14px;
  padding-bottom: 8px;
  border-bottom: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
.color-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
  flex-wrap: wrap;
  gap: 8px;
}
.color-row-label { font-size: 12px; color: #556677; min-width: 80px; }
.color-row-right { display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
.swatches { display: flex; gap: 5px; }
.swatch {
  width: 24px; height: 24px; border-radius: 50%;
  border: 2px solid rgba(0,0,0,0.08);
  cursor: pointer; flex-shrink: 0;
  transition: transform 0.1s, box-shadow 0.1s;
}
.swatch:hover { transform: scale(1.15); }
.swatch.active { box-shadow: 0 0 0 2px white, 0 0 0 4px #446; }
.hex-input {
  width: 80px; height: 28px;
  border: 1.5px solid var(--color-border);
  border-radius: 8px;
  padding: 0 8px;
  font-size: 11px;
  color: #334455;
  font-family: monospace;
  outline: none;
}
.hex-input:focus { border-color: var(--color-accent); }
.save-theme-btn {
  width: 100%;
  margin-top: 14px;
  padding: 11px;
  background: var(--color-accent);
  border: none;
  border-radius: 12px;
  font-size: 13px;
  font-weight: bold;
  color: white;
  cursor: pointer;
  font-family: inherit;
}
.save-theme-btn:hover { opacity: 0.88; }
.logout-btn {
  width: 100%;
  padding: 10px;
  background: none;
  border: 1.5px solid #dde;
  border-radius: 10px;
  font-size: 13px;
  color: #889;
  cursor: pointer;
  font-family: inherit;
}
.logout-btn:hover { border-color: #e57373; color: #e57373; }

/* 통계 */
.stats-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
  padding: 12px 14px 0;
}
.stats-reset-btn {
  background: none;
  border: 1px solid #dde;
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 11px;
  color: #bbb;
  cursor: pointer;
  font-family: inherit;
}
.stats-reset-btn:hover { color: #e57373; border-color: #e57373; }
.streak-card {
  background: color-mix(in srgb, var(--color-bg) 20%, white);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 14px;
  display: flex;
  align-items: center;
  margin: 0 14px 16px;
}
.streak-item { flex: 1; text-align: center; }
.streak-num { font-size: 22px; font-weight: 800; color: var(--color-accent); line-height: 1.2; }
.streak-label { font-size: 11px; color: #aabbcc; margin-top: 2px; }
.streak-divider { width: 1px; background: var(--color-border); height: 36px; flex-shrink: 0; }
.stats-section-title {
  font-size: 12px;
  font-weight: 700;
  color: var(--color-accent);
  margin-bottom: 8px;
  padding: 0 14px;
}
.stats-cal-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 3px;
  text-align: center;
  margin-bottom: 8px;
  padding: 0 14px;
}
.stats-cal-header { font-size: 10px; color: #aaa; padding: 2px 0; }
.stats-cal-day {
  width: 28px; height: 28px;
  border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  font-size: 10px;
  margin: 0 auto;
}
.stats-cal-day.done { background: color-mix(in srgb, var(--color-bg) 80%, white); color: var(--color-accent); }
.stats-cal-day.missed { background: #f5f5f5; color: #ccc; }
.stats-cal-day.today { background: color-mix(in srgb, var(--color-bg) 40%, white); border: 2px solid var(--color-border); color: var(--color-accent); font-weight: 700; }
.stats-cal-day.future { color: #ddd; }
.stats-cal-legend {
  display: flex; gap: 10px;
  font-size: 11px; color: #aaa;
  margin-bottom: 16px;
  padding: 0 14px;
}
.stats-cal-legend span { display: flex; align-items: center; gap: 4px; }
.legend-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
.rate-row { margin-bottom: 8px; padding: 0 14px; }
.rate-header { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 3px; }
.rate-bar-bg { background: #eee; border-radius: 4px; height: 7px; }
.rate-bar-fill { height: 7px; border-radius: 4px; }
.rate-bar-fill.good { background: color-mix(in srgb, var(--color-bg) 80%, white); }
.rate-bar-fill.low  { background: #ffc4a0; }
.full-reset-area {
  margin-top: 20px;
  padding: 14px 14px 14px;
  border-top: 1px solid color-mix(in srgb, var(--color-border) 30%, white);
  text-align: center;
}
.full-reset-btn {
  background: none; border: none;
  font-size: 12px; color: #ccc;
  cursor: pointer; font-family: inherit;
}
.full-reset-btn:hover { color: #e57373; }
```

- [ ] **Step 2: 브라우저에서 전체 확인**

앱을 열어 다음을 확인:
- 로그인 화면: 흰 카드에 하늘색 테두리
- 메인 화면: 보케 배경, 싸이월드 헤더, 탭바, 루틴 아이템 스타일
- 통계 탭: 스트릭 카드, 달력, 달성률 바 스타일
- 스케줄 화면: 달력 카드 스타일

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat: 전체 CSS 싸이월드 스타일 적용"
```

---

### Task 4: 설정 화면 색상 피커 구현

**Files:**
- Modify: `index.html` — `renderSettings()` 함수 완성

Task 2에서 추가한 빈 `renderSettings()` 함수를 완성한다.

- [ ] **Step 1: 색상 피커 상수 정의 추가**

`DEFAULT_THEME` 상수 바로 아래에:

```javascript
const COLOR_PICKER_CONFIG = [
  {
    key: 'bg',
    label: '🌈 배경 색',
    swatches: ['#b2e0f7','#ffd0e8','#c8f0d8','#f0d8f8','#ffeec8'],
  },
  {
    key: 'border',
    label: '🖼 테두리 색',
    swatches: ['#88ccee','#ffaad4','#88ccb8','#cc99ee','#ffbb88'],
  },
  {
    key: 'accent',
    label: '✨ 포인트 색',
    swatches: ['#4488bb','#cc4477','#339977','#8855cc','#cc7733'],
  },
  {
    key: 'card',
    label: '🗂 카드 배경',
    swatches: ['#ffffff','#f8fdff','#fff8fc','#f8fff8','#fffaf4'],
  },
];
```

- [ ] **Step 2: `renderSettings()` 함수 완성**

기존 빈 `renderSettings()` 함수를 아래로 교체:

```javascript
function renderSettings() {
  const theme = loadTheme();
  const container = document.getElementById('color-pickers');
  container.innerHTML = '';

  COLOR_PICKER_CONFIG.forEach(({ key, label, swatches }) => {
    const row = document.createElement('div');
    row.className = 'color-row';

    const swatchesHtml = swatches.map(color => `
      <div class="swatch ${theme[key] === color ? 'active' : ''}"
           style="background:${color}"
           data-key="${key}" data-color="${color}"
           onclick="onSwatchClick(this)"></div>
    `).join('');

    row.innerHTML = `
      <span class="color-row-label">${label}</span>
      <div class="color-row-right">
        <div class="swatches">${swatchesHtml}</div>
        <input class="hex-input" id="hex-${key}" value="${theme[key]}"
               oninput="onHexInput(this, '${key}')" maxlength="7" />
      </div>
    `;
    container.appendChild(row);
  });
}
```

- [ ] **Step 3: `onSwatchClick()`, `onHexInput()`, `onSaveTheme()` 함수 추가**

`renderSettings` 바로 아래에:

```javascript
function onSwatchClick(el) {
  const key = el.dataset.key;
  const color = el.dataset.color;
  // 같은 key의 active 제거
  document.querySelectorAll(`.swatch[data-key="${key}"]`).forEach(s => s.classList.remove('active'));
  el.classList.add('active');
  document.getElementById('hex-' + key).value = color;
}

function onHexInput(el, key) {
  const val = el.value.trim();
  if (/^#[0-9a-fA-F]{6}$/.test(val)) {
    document.querySelectorAll(`.swatch[data-key="${key}"]`).forEach(s => s.classList.remove('active'));
  }
}

function onSaveTheme() {
  const theme = {};
  COLOR_PICKER_CONFIG.forEach(({ key }) => {
    const val = document.getElementById('hex-' + key).value.trim();
    theme[key] = /^#[0-9a-fA-F]{6}$/.test(val) ? val : DEFAULT_THEME[key];
  });
  saveTheme(theme);
  showScreen('main');
}
```

- [ ] **Step 4: 브라우저에서 색상 피커 확인**

설정 화면 진입 → 스와치 클릭 → 헥스 입력 → 저장 → 앱 전체 색상 변경 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: 설정 화면 색상 피커 구현"
```

---

### Task 5: 헤더 스트릭 뱃지 + 마무리 정리

**Files:**
- Modify: `index.html` — renderMain() 스트릭 뱃지 업데이트 + 잔여 코드 정리

- [ ] **Step 1: `renderMain()` 끝에 스트릭 뱃지 비동기 업데이트 추가**

`renderMain()` 함수 맨 끝(return 전)에 추가:

```javascript
// 헤더 스트릭 뱃지 비동기 업데이트
updateStreakBadge();
```

- [ ] **Step 2: `updateStreakBadge()` 함수 추가**

`renderMain` 함수 바로 앞에:

```javascript
async function updateStreakBadge() {
  const badge = document.getElementById('streak-header-badge');
  if (!badge) return;
  try {
    const allChecks = await loadAllRoutineChecks();
    const { current } = calcStreakInfo(allChecks, currentRoutines, statsYear, statsMonth);
    if (current > 0) {
      badge.textContent = `🔥 ${current}일 연속`;
      badge.style.display = '';
    } else {
      badge.style.display = 'none';
    }
  } catch {
    badge.style.display = 'none';
  }
}
```

- [ ] **Step 3: 브라우저에서 로그인 후 헤더 스트릭 뱃지 확인**

루틴을 이미 며칠간 완료한 경우 "🔥 N일 연속" 뱃지 노출 확인. 0일이면 숨김 확인.

- [ ] **Step 4: 전체 화면 최종 점검**

다음 시나리오 모두 확인:
1. 로그인 화면 → Google 로그인 → 메인 화면 정상 렌더
2. 루틴 체크/해제 동작
3. 루틴 편집(추가/삭제/드래그 순서변경) 동작
4. 통계 탭 → 스트릭/달력/달성률 렌더
5. 스케줄 화면 → ← 뒤로 정상 동작
6. 설정 화면 → 색상 변경 → 저장 → 즉시 반영 확인
7. 페이지 새로고침 후 색상 설정 유지 확인

- [ ] **Step 5: 최종 커밋 + 배포**

```bash
git add index.html
git commit -m "feat: 싸이월드 미니홈피 리디자인 완료 - 헤더 스트릭 뱃지 + 최종 정리"
git push
```

---

## 주의사항

- `color-mix(in srgb, ...)` 함수는 Chrome 111+, Safari 16.2+ 지원. 구형 브라우저 미지원 시 fallback 없이 색이 안 보일 수 있으나, 모바일 위주 앱이므로 허용.
- `#screen-schedule`의 기존 `.header` → `.sub-screen-header`로 클래스명이 바뀌므로 다른 JS에서 `.header` 선택자 사용 중인지 확인 (없음 — JS는 id로만 참조).
- Task 3의 CSS 교체 시 기존 `<style>` 태그 내용 전체를 교체하되, `<style>`과 `</style>` 태그 자체는 유지.
