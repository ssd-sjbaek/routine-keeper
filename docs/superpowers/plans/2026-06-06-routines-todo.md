# 루틴 편집 + 오늘의 할 일 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 기존 루틴 앱에 루틴 편집(추가/삭제)과 매일 다른 할 일을 관리하는 오늘의 할 일 섹션을 추가한다.

**Architecture:** `index.html` 단일 파일만 수정. Firestore에 `/users/{uid}/settings/app`(루틴 목록), `/users/{uid}/todos/list`(할 일 목록) 문서를 추가. 루틴은 이모지 포함 단일 문자열 배열로 저장. 루틴 편집은 인라인 편집 모드, 할 일 추가는 인라인 입력 필드로 구현.

**Tech Stack:** Firebase compat SDK 10.7.1 (기존), Firestore, vanilla JS, 단일 HTML 파일

---

### Task 1: CSS + HTML 구조 변경

**Files:**
- Modify: `index.html`

- [ ] **Step 1: `</style>` 바로 위에 CSS 추가**

`index.html`의 `  </style>` (213번째 줄) 바로 위에 아래 CSS를 삽입한다:

```css
    /* 섹션 헤더 (루틴 + 할 일 공통) */
    .section-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 8px;
    }
    .section-title {
      font-size: 12px;
      font-weight: 700;
      color: #88a89c;
    }
    .section-btn {
      background: none;
      border: none;
      color: #2e7d5e;
      font-size: 13px;
      cursor: pointer;
      padding: 2px 4px;
    }
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
    .add-input-row {
      display: flex;
      gap: 8px;
      margin-top: 8px;
    }
    .add-input {
      flex: 1;
      border: 1px solid #d4ede4;
      border-radius: 8px;
      padding: 8px 12px;
      font-size: 14px;
      outline: none;
      font-family: inherit;
    }
    .add-input:focus { border-color: #2e7d5e; }
    .add-btn {
      background: #2e7d5e;
      color: white;
      border: none;
      border-radius: 8px;
      padding: 8px 14px;
      font-size: 14px;
      cursor: pointer;
    }
    .add-btn:hover { background: #245f49; }
    /* 할 일 섹션 */
    .todo-section {
      margin-top: 16px;
      padding-top: 16px;
      border-top: 1px solid #f0f0f0;
    }
    .todo-item {
      display: flex;
      align-items: center;
      gap: 10px;
      background: #f9f9f9;
      border-radius: 10px;
      padding: 12px;
      font-size: 15px;
      margin-bottom: 8px;
    }
    .todo-item.done .todo-text {
      text-decoration: line-through;
      color: #aaa;
    }
    .todo-item input[type="checkbox"] {
      width: 18px;
      height: 18px;
      accent-color: #2e7d5e;
      cursor: pointer;
      flex-shrink: 0;
    }
    .todo-text { flex: 1; }
```

- [ ] **Step 2: 메인 카드 HTML 수정**

249~250번째 줄의 아래 두 줄을:
```html
      <div id="routine-list" class="routine-list"></div>
      <div id="progress" class="progress">0 / 5 완료</div>
```

아래로 교체한다:
```html
      <div id="routine-section" class="routine-list"></div>
      <div id="todo-section" class="todo-section"></div>
```

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "style: 루틴 편집 + 할 일 섹션 CSS/HTML 기반 추가"
```

---

### Task 2: 루틴 데이터 레이어

**Files:**
- Modify: `index.html` (`<script>` 블록)

- [ ] **Step 1: `ROUTINES` 상수 블록 교체**

300~306번째 줄의 아래 블록을:
```javascript
    const ROUTINES = [
      { name: '환기',             icon: '🪟' },
      { name: '향 피우기',        icon: '🕯️' },
      { name: '이불 정리',        icon: '🛏️' },
      { name: '청소기 돌리기',    icon: '🧹' },
      { name: '콩이 화장실 정리', icon: '🐱' },
    ];
```

아래로 교체한다:
```javascript
    const DEFAULT_ROUTINES = [
      "🪟 환기",
      "🕯️ 향 피우기",
      "🛏️ 이불 정리",
      "🧹 청소기 돌리기",
      "🐱 콩이 화장실 정리",
    ];

    let currentRoutines = [];
    let routineEditMode = false;
    let todoAddMode = false;
```

- [ ] **Step 2: Firestore 루틴 설정 함수 추가**

`loadRoutineChecks()` 함수(381번째 줄) 바로 아래에 추가한다:

```javascript
    async function loadRoutineList() {
      const doc = await db.collection('users').doc(currentUser.uid)
                           .collection('settings').doc('app').get();
      return doc.exists ? doc.data().routines : null;
    }

    async function saveRoutineList(routines) {
      await db.collection('users').doc(currentUser.uid)
              .collection('settings').doc('app')
              .set({ routines }, { merge: true });
    }

    async function initRoutineSettings() {
      const existing = await loadRoutineList();
      if (existing !== null) {
        currentRoutines = existing;
      } else {
        currentRoutines = [...DEFAULT_ROUTINES];
        await saveRoutineList(currentRoutines);
      }
    }
```

- [ ] **Step 3: `auth.onAuthStateChanged` 업데이트**

아래 기존 코드를:
```javascript
    auth.onAuthStateChanged(async (user) => {
      currentUser = user;
      if (user) {
        await migrateFromLocalStorage();
        showScreen('main');
      } else {
        showScreen('login');
      }
    });
```

아래로 교체한다:
```javascript
    auth.onAuthStateChanged(async (user) => {
      currentUser = user;
      if (user) {
        await migrateFromLocalStorage();
        await initRoutineSettings();
        showScreen('main');
      } else {
        routineEditMode = false;
        todoAddMode = false;
        showScreen('login');
      }
    });
```

- [ ] **Step 4: 브라우저에서 확인**

로그아웃 후 재로그인. Firebase Console → Firestore → `users/{uid}/settings/app` 문서가 생성됐고 `routines` 필드에 5개 문자열 배열이 있는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: 루틴 목록 Firestore 저장/로드 추가"
```

---

### Task 3: 루틴 편집 함수 추가

**Files:**
- Modify: `index.html` (`<script>` 블록)

- [ ] **Step 1: 루틴 편집 함수 추가**

`toggleRoutine()` 함수 바로 아래에 추가한다:

```javascript
    async function toggleRoutineEdit() {
      if (routineEditMode) {
        routineEditMode = false;
        await saveRoutineList(currentRoutines);
      } else {
        routineEditMode = true;
      }
      await renderMain();
    }

    async function deleteRoutine(index) {
      currentRoutines.splice(index, 1);
      await renderMain();
    }

    async function addRoutine() {
      const input = document.getElementById('new-routine-input');
      const name = input.value.trim();
      if (!name) return;
      currentRoutines.push(name);
      input.value = '';
      await renderMain();
      const newInput = document.getElementById('new-routine-input');
      if (newInput) newInput.focus();
    }
```

- [ ] **Step 2: 커밋**

```bash
git add index.html
git commit -m "feat: 루틴 편집(추가/삭제) 함수 추가"
```

---

### Task 4: 할 일 데이터 + CRUD 함수 추가

**Files:**
- Modify: `index.html` (`<script>` 블록)

- [ ] **Step 1: Firestore 할 일 함수 추가**

`initRoutineSettings()` 함수 바로 아래에 추가한다:

```javascript
    async function loadTodoList() {
      const doc = await db.collection('users').doc(currentUser.uid)
                           .collection('todos').doc('list').get();
      return doc.exists ? (doc.data().items || []) : [];
    }

    async function saveTodoList(items) {
      await db.collection('users').doc(currentUser.uid)
              .collection('todos').doc('list')
              .set({ items });
    }
```

- [ ] **Step 2: 할 일 CRUD 함수 추가**

`addRoutine()` 함수 바로 아래에 추가한다:

```javascript
    function showTodoInput() {
      todoAddMode = true;
      renderMain();
    }

    async function addTodo() {
      const input = document.getElementById('new-todo-input');
      const text = input.value.trim();
      if (!text) return;
      const items = await loadTodoList();
      items.push({
        id: crypto.randomUUID(),
        text,
        done: false,
        doneDate: null,
      });
      await saveTodoList(items);
      todoAddMode = false;
      await renderMain();
    }

    async function toggleTodo(id, checked) {
      const today = getTodayKey();
      const items = await loadTodoList();
      const item = items.find(i => i.id === id);
      if (item) {
        item.done = checked;
        item.doneDate = checked ? today : null;
      }
      await saveTodoList(items);
      await renderMain();
    }

    async function deleteTodo(id) {
      const items = await loadTodoList();
      await saveTodoList(items.filter(i => i.id !== id));
      await renderMain();
    }
```

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat: 오늘의 할 일 Firestore 함수 추가"
```

---

### Task 5: renderMain() 전체 교체

**Files:**
- Modify: `index.html` (`<script>` 블록)

- [ ] **Step 1: `renderMain()` 전체 교체**

기존 `async function renderMain()` 함수 전체(408~443번째 줄)를 아래로 교체한다:

```javascript
    async function renderMain() {
      const today = getTodayKey();
      const [shift, checks, allTodos] = await Promise.all([
        loadShift(today),
        loadRoutineChecks(today),
        loadTodoList(),
      ]);

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

      // ── 루틴 섹션 ──
      const routineSection = document.getElementById('routine-section');
      routineSection.innerHTML = '';

      const routineHeader = document.createElement('div');
      routineHeader.className = 'section-header';
      routineHeader.innerHTML = `
        <span class="section-title">✅ 매일 루틴</span>
        <button class="section-btn" onclick="toggleRoutineEdit()">
          ${routineEditMode ? '완료' : '편집'}
        </button>
      `;
      routineSection.appendChild(routineHeader);

      currentRoutines.forEach((name, index) => {
        if (routineEditMode) {
          const item = document.createElement('div');
          item.className = 'routine-item';
          item.innerHTML = `
            <span class="routine-name">${name}</span>
            <button class="routine-delete" onclick="deleteRoutine(${index})">✕</button>
          `;
          routineSection.appendChild(item);
        } else {
          const isChecked = checks[name] || false;
          const item = document.createElement('label');
          item.className = 'routine-item' + (isChecked ? ' done' : '');
          item.innerHTML = `
            <input type="checkbox" ${isChecked ? 'checked' : ''}
                   onchange="toggleRoutine('${name}', this.checked)">
            <span class="routine-name">${name}</span>
          `;
          routineSection.appendChild(item);
        }
      });

      if (routineEditMode) {
        const addRow = document.createElement('div');
        addRow.className = 'add-input-row';
        addRow.innerHTML = `
          <input class="add-input" id="new-routine-input" placeholder="새 루틴 입력..."
                 onkeydown="if(event.key==='Enter')addRoutine()">
          <button class="add-btn" onclick="addRoutine()">추가</button>
        `;
        routineSection.appendChild(addRow);
      }

      const progressEl = document.createElement('div');
      progressEl.className = 'progress';
      const routineDoneCount = currentRoutines.filter(name => checks[name]).length;
      progressEl.textContent = `${routineDoneCount} / ${currentRoutines.length} 완료`;
      routineSection.appendChild(progressEl);

      // ── 할 일 섹션 ──
      const visibleTodos = allTodos.filter(t => !t.done || t.doneDate === today);
      if (visibleTodos.length !== allTodos.length) {
        await saveTodoList(visibleTodos);
      }

      const todoSection = document.getElementById('todo-section');
      todoSection.innerHTML = '';

      const todoHeader = document.createElement('div');
      todoHeader.className = 'section-header';
      todoHeader.innerHTML = `
        <span class="section-title">📋 오늘의 할 일</span>
        <button class="section-btn" onclick="showTodoInput()">+</button>
      `;
      todoSection.appendChild(todoHeader);

      visibleTodos.forEach(item => {
        const el = document.createElement('div');
        el.className = 'todo-item' + (item.done ? ' done' : '');
        el.innerHTML = `
          <input type="checkbox" ${item.done ? 'checked' : ''}
                 onchange="toggleTodo('${item.id}', this.checked)">
          <span class="todo-text">${item.text}</span>
          <button class="todo-delete" onclick="deleteTodo('${item.id}')">✕</button>
        `;
        todoSection.appendChild(el);
      });

      if (todoAddMode) {
        const addRow = document.createElement('div');
        addRow.className = 'add-input-row';
        addRow.innerHTML = `
          <input class="add-input" id="new-todo-input" placeholder="할 일 입력..."
                 onkeydown="if(event.key==='Enter')addTodo()">
          <button class="add-btn" onclick="addTodo()">추가</button>
        `;
        todoSection.appendChild(addRow);
        setTimeout(() => {
          const el = document.getElementById('new-todo-input');
          if (el) el.focus();
        }, 50);
      }
    }
```

- [ ] **Step 2: 브라우저에서 전체 기능 확인**

로컬에서 `index.html`을 브라우저로 열고 (로그인 후) 아래 항목 확인:

1. 메인 화면에 "✅ 매일 루틴" 섹션 + "편집" 버튼 보임
2. "편집" 클릭 → 각 루틴 옆 ✕ 버튼 나타남 + 입력 필드 나타남
3. 새 루틴 입력 + "추가" 클릭 → 루틴 추가됨
4. ✕ 클릭 → 루틴 삭제됨
5. "완료" 클릭 → 편집 모드 종료 + Firestore에 저장됨 (Firebase Console 확인)
6. 메인 화면에 "📋 오늘의 할 일" 섹션 + "+" 버튼 보임
7. "+" 클릭 → 입력 필드 나타남
8. 할 일 입력 후 엔터 → 목록에 추가됨
9. 체크박스 클릭 → 줄 그어짐 (완료 표시)
10. ✕ 클릭 → 항목 삭제됨
11. Firebase Console에서 `users/{uid}/todos/list` 문서의 `items` 배열 확인

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat: renderMain 전체 교체 - 루틴 편집 + 할 일 섹션 완성"
```

---

### Task 6: 배포 + 검증

**Files:**
- 없음 (배포만)

- [ ] **Step 1: GitHub에 푸시**

```bash
git push
```

- [ ] **Step 2: 배포 완료 대기**

`https://github.com/ssd-sjbaek/routine-keeper/actions` 에서 "pages build and deployment" 완료 확인 (약 1~2분 소요)

- [ ] **Step 3: 크로스 디바이스 테스트**

`https://ssd-sjbaek.github.io/routine-keeper/` 에서:

1. 컴퓨터에서 새 루틴 추가 ("🏃 달리기" 등) → 폰에서 새로고침 → 동일하게 보이는지 확인
2. 폰에서 할 일 추가 → 컴퓨터에서 새로고침 → 동일하게 보이는지 확인
3. 완료된 할 일이 다음 날 사라지는지는 날짜를 바꿔서 확인하기 어려우므로, Firestore Console에서 `doneDate`가 오늘 날짜로 저장됐는지 확인

- [ ] **Step 4: 완료**

모든 기능 작동 확인됨.
