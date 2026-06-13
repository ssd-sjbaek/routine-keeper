# 근무 스티커 + 프로필 커스텀 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 캘린더 날짜 셀에서 A/C/D 근무를 탭으로 지정하고 홈 화면에 표시하며, 설정 탭에서 앱 이름과 닉네임을 변경할 수 있게 한다.

**Architecture:** 단일 HTML 파일(`index.html`)에 CSS, HTML, JS가 모두 있다. Firebase Firestore compat SDK 10.7.1 사용. 근무 데이터는 `users/{uid}/shifts/{YYYY-MM-DD}` 경로에 document ID = dateKey로 저장. 프로필(`appName`, `nickname`)은 기존 `users/{uid}` 문서에 필드 추가.

**Tech Stack:** Vanilla JS, Firebase Firestore compat SDK 10.7.1 (CDN), CSS Custom Properties

---

## 파일 구조

모든 변경은 `index.html` 한 파일에 집중된다.

| 변경 위치 | 내용 |
|-----------|------|
| CSS `<style>` (~line 599 이후) | `.shift-badge`, `.shift-badge-A/C/D`, `.date-hint-shift` CSS 추가 |
| JS — Firestore 함수 블록 (~line 1075 이후) | `loadShiftsForMonth`, `loadTodayShift`, `toggleShift` 추가 |
| JS — `renderCalendar()` (~line 1664) | shifts 동시 로드, 셀에 스티커 뱃지 + 클릭 분리 |
| JS — `renderMain()` (~line 1250) | 오늘 shift 로드 후 date-hint-bar에 뱃지 표시 |
| HTML — 설정 화면 (~line 821) | 프로필 `.settings-card` 섹션 추가 |
| JS — Firestore 함수 블록 (~line 1075 이후) | `loadUserProfile`, `saveAppName`, `saveNickname` 추가 |
| JS — `renderSettings()` (~line 1610) | 프로필 필드 초기화 코드 추가 |
| JS — `renderMain()` (~line 1259) | appName/nickname Firestore 값으로 헤더 업데이트 |
| JS — `onAuthStateChanged` (~line 1885) | 로그인 시 `loadUserProfile()` 호출 추가 |

---

## Task 1: Shift Firestore 함수 추가

**Files:**
- Modify: `index.html` (JS section, `deleteEvent()` 함수 끝 ~line 1075 이후)

- [ ] **Step 1: `deleteEvent()` 함수 끝(line ~1075) 바로 뒤에 다음 세 함수를 삽입한다**

```javascript
    // ── 근무 스티커 Firestore 함수 ────────────────────────
    async function loadShiftsForMonth(year, month) {
      const pad = n => String(n).padStart(2, '0');
      const start   = `${year}-${pad(month)}-01`;
      const lastDay = new Date(year, month, 0).getDate();
      const end     = `${year}-${pad(month)}-${pad(lastDay)}`;
      const snap = await db.collection('users').doc(currentUser.uid)
        .collection('shifts')
        .where('date', '>=', start)
        .where('date', '<=', end)
        .get();
      const result = {};
      snap.forEach(doc => { result[doc.data().date] = doc.data().shift; });
      return result;
    }

    async function loadTodayShift() {
      const doc = await db.collection('users').doc(currentUser.uid)
        .collection('shifts').doc(getTodayKey()).get();
      return doc.exists ? doc.data().shift : null;
    }

    async function toggleShift(dateKey) {
      const ref = db.collection('users').doc(currentUser.uid)
        .collection('shifts').doc(dateKey);
      const doc = await ref.get();
      const order = ['A', 'C', 'D'];
      if (!doc.exists) {
        await ref.set({ date: dateKey, shift: 'A', createdAt: firebase.firestore.FieldValue.serverTimestamp() });
      } else {
        const current = doc.data().shift;
        const idx = order.indexOf(current);
        if (idx === order.length - 1) {
          await ref.delete();
        } else {
          await ref.update({ shift: order[idx + 1] });
        }
      }
    }
```

- [ ] **Step 2: 브라우저 콘솔에서 수동 검증**

로그인된 상태에서 브라우저 DevTools 콘솔에 입력:

```javascript
// 오늘 날짜에 shift 토글 (없음 → A → C → D → 없음)
await toggleShift('2026-06-13');
// Firestore 콘솔에서 users/{uid}/shifts/2026-06-13 문서 확인
// shift: "A" 가 저장되어 있어야 함

await toggleShift('2026-06-13');
// shift: "C" 로 업데이트되어야 함

const s = await loadTodayShift();
console.log(s); // "C" 출력되어야 함

const month = await loadShiftsForMonth(2026, 6);
console.log(month); // { "2026-06-13": "C" } 포함되어야 함
```

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "feat: 근무 스티커 Firestore 함수 추가 (loadShiftsForMonth, loadTodayShift, toggleShift)"
```

---

## Task 2: 근무 스티커 CSS + 캘린더 렌더링 수정

**Files:**
- Modify: `index.html` (CSS section ~line 599, JS `renderCalendar()` ~line 1664)

- [ ] **Step 1: `.event-more` CSS 규칙 끝(line ~599) 바로 뒤에 shift badge CSS를 추가한다**

```css
    /* 근무 스티커 */
    .shift-badge {
      font-size: 9px;
      font-weight: 700;
      border-radius: 10px;
      padding: 1px 5px;
      line-height: 1.4;
      cursor: pointer;
      display: inline-block;
      user-select: none;
    }
    .shift-badge-A { background: #fde8e0; color: #c05030; }
    .shift-badge-C { background: #e8f0d0; color: #5a7a3a; }
    .shift-badge-D { background: #ece8f8; color: #5040a0; }
    .shift-badge-empty {
      font-size: 8px;
      color: #ddeeff;
      border-radius: 10px;
      padding: 1px 5px;
      line-height: 1.4;
      cursor: pointer;
      display: inline-block;
      user-select: none;
      border: 1px dashed #ccdde8;
    }
```

- [ ] **Step 2: `renderCalendar()` 함수를 수정한다 (line ~1664)**

`renderCalendar()` 내부에서 `const events = await loadEventsForMonth(year, month);` 한 줄을 다음으로 교체한다:

```javascript
        const [events, shifts] = await Promise.all([
          loadEventsForMonth(year, month),
          loadShiftsForMonth(year, month),
        ]);
```

- [ ] **Step 3: 셀 렌더링 루프에서 shift 뱃지와 클릭 분리를 적용한다**

`renderCalendar()` 내부의 `cell.onclick = () => openDayModal(dateKey);` 한 줄을 다음으로 교체한다:

```javascript
          numSpan.style.cursor = 'pointer';
          numSpan.onclick = (e) => { e.stopPropagation(); openDayModal(dateKey); };

          const shiftVal = shifts[dateKey];
          const shiftEl = document.createElement('span');
          if (shiftVal) {
            shiftEl.className = `shift-badge shift-badge-${escapeHtml(shiftVal)}`;
            shiftEl.textContent = shiftVal;
          } else {
            shiftEl.className = 'shift-badge-empty';
            shiftEl.textContent = '+';
          }
          shiftEl.onclick = async (e) => {
            e.stopPropagation();
            try {
              await toggleShift(dateKey);
              await renderCalendar();
            } catch {
              alert('근무 저장에 실패했어요. 다시 시도해주세요.');
            }
          };
          cell.appendChild(shiftEl);
```

또한 같은 루프에서 `cell.onclick = () => openDayModal(dateKey);` 라인을 완전히 제거한다 (numSpan.onclick으로 대체됐으므로).

- [ ] **Step 4: 캘린더의 `.cal-day` CSS에서 `cursor: pointer` 제거**

`cursor: pointer;` 줄을 `cursor: default;` 로 교체한다 (셀 전체가 아니라 숫자와 뱃지에만 pointer 적용).

```css
    .cal-day {
      background: var(--color-card);
      border: 1px solid color-mix(in srgb, var(--color-border) 40%, white);
      border-radius: 8px;
      padding: 4px 3px 5px;
      text-align: center;
      cursor: default;
      min-height: 48px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 2px;
      box-shadow: 0 1px 3px rgba(100,160,220,0.07);
      transition: background 0.1s;
      overflow: hidden;
    }
```

- [ ] **Step 5: 브라우저에서 캘린더 화면 수동 검증**

1. 스케줄 탭으로 이동
2. 날짜 셀에 `+` 스티커 영역이 보이는지 확인
3. `+` 탭 → A 뱃지(붉은색)로 바뀌는지 확인
4. 다시 탭 → C(초록), D(보라), 없음(+) 순서로 순환하는지 확인
5. 날짜 숫자 탭 → 일정 모달이 열리는지 확인

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "feat: 캘린더 근무 스티커 표시 + 탭 토글 기능 추가"
```

---

## Task 3: 홈 화면 오늘 근무 뱃지 표시

**Files:**
- Modify: `index.html` (CSS ~line 164, JS `renderMain()` ~line 1257)

- [ ] **Step 1: `.date-hint-bar` CSS 뒤에 홈용 shift 뱃지 CSS를 추가한다 (~line 164)**

`.date-hint-bar` 규칙 바로 뒤에 추가:

```css
    .date-hint-shift {
      display: inline-flex;
      align-items: center;
      gap: 6px;
    }
    .date-hint-shift .shift-badge {
      font-size: 10px;
      padding: 1px 7px;
    }
```

- [ ] **Step 2: `renderMain()` 에서 date-hint-bar 갱신 로직을 수정한다 (~line 1250)**

기존 코드:
```javascript
      document.getElementById('date-hint-bar').textContent = getTodayLabel();
```

이 한 줄을 다음으로 교체한다:

```javascript
      let todayShift = null;
      try { todayShift = await loadTodayShift(); } catch { /* 표시 생략 */ }
      const shiftHtml = todayShift
        ? `<span class="shift-badge shift-badge-${todayShift}">${escapeHtml(todayShift)}</span>`
        : '';
      document.getElementById('date-hint-bar').innerHTML =
        `<span class="date-hint-shift">${escapeHtml(getTodayLabel())}${shiftHtml}</span>`;
```

- [ ] **Step 3: 브라우저에서 홈 화면 수동 검증**

1. 홈 탭으로 이동
2. Task 1에서 설정한 오늘 날짜(2026-06-13)의 shift가 날짜 텍스트 옆에 뱃지로 표시되는지 확인
3. 캘린더에서 오늘 날짜 shift를 없음으로 바꾼 뒤 홈으로 돌아오면 뱃지가 사라지는지 확인
   (홈 탭 클릭 시 `renderMain()` 재호출되어 반영됨)

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "feat: 홈 화면 오늘 날짜 옆 근무 뱃지 표시"
```

---

## Task 4: 프로필 커스텀 (Firestore + 설정 UI + 홈 반영)

**Files:**
- Modify: `index.html` (JS Firestore 함수 블록, 설정 HTML ~line 821, `renderSettings()` ~line 1610, `renderMain()` ~line 1259, `onAuthStateChanged` ~line 1885)

- [ ] **Step 1: Task 1에서 추가한 shift 함수 블록 바로 뒤에 프로필 Firestore 함수 3개를 삽입한다**

```javascript
    // ── 프로필 Firestore 함수 ─────────────────────────
    let cachedProfile = null;

    async function loadUserProfile() {
      const doc = await db.collection('users').doc(currentUser.uid).get();
      cachedProfile = doc.exists ? { appName: doc.data().appName || '', nickname: doc.data().nickname || '' } : { appName: '', nickname: '' };
      return cachedProfile;
    }

    async function saveAppName(name) {
      await db.collection('users').doc(currentUser.uid).set({ appName: name }, { merge: true });
      if (cachedProfile) cachedProfile.appName = name;
    }

    async function saveNickname(name) {
      await db.collection('users').doc(currentUser.uid).set({ nickname: name }, { merge: true });
      if (cachedProfile) cachedProfile.nickname = name;
    }
```

- [ ] **Step 2: 설정 화면 HTML에 프로필 섹션을 추가한다 (~line 821)**

기존 설정 화면 HTML에서 색상 카드(`<div class="settings-card">...색상 꾸미기...`) 바로 위에 삽입:

```html
    <div class="settings-card" style="margin-bottom:12px">
      <div class="settings-section-title">👤 프로필</div>
      <div class="profile-field-row">
        <label class="profile-field-label">앱 이름</label>
        <div class="profile-field-input-wrap">
          <input id="input-app-name" class="profile-input" type="text" maxlength="20" placeholder="루틴 지킴이">
          <button class="profile-save-btn" onclick="onSaveAppName()">저장</button>
        </div>
      </div>
      <div class="profile-field-row">
        <label class="profile-field-label">닉네임</label>
        <div class="profile-field-input-wrap">
          <input id="input-nickname" class="profile-input" type="text" maxlength="20" placeholder="나의 홈">
          <button class="profile-save-btn" onclick="onSaveNickname()">저장</button>
        </div>
      </div>
    </div>
```

- [ ] **Step 3: 프로필 섹션 CSS를 `.event-more` 규칙 뒤(또는 shift badge CSS 뒤)에 추가한다**

```css
    /* 프로필 설정 */
    .profile-field-row {
      display: flex;
      flex-direction: column;
      gap: 4px;
      margin-bottom: 12px;
    }
    .profile-field-row:last-child { margin-bottom: 0; }
    .profile-field-label {
      font-size: 11px;
      color: #99bbcc;
    }
    .profile-field-input-wrap {
      display: flex;
      gap: 8px;
      align-items: center;
    }
    .profile-input {
      flex: 1;
      border: 1.5px solid var(--color-border);
      border-radius: 8px;
      padding: 7px 10px;
      font-size: 13px;
      font-family: inherit;
      color: #334455;
      background: var(--color-card);
    }
    .profile-input:focus { outline: none; border-color: var(--color-accent); }
    .profile-save-btn {
      background: color-mix(in srgb, var(--color-bg) 30%, white);
      border: 1px solid var(--color-border);
      border-radius: 8px;
      padding: 7px 12px;
      font-size: 12px;
      color: var(--color-accent);
      cursor: pointer;
      font-family: inherit;
      white-space: nowrap;
    }
    .profile-save-btn:hover { background: color-mix(in srgb, var(--color-bg) 50%, white); }
```

- [ ] **Step 4: `renderSettings()` 함수 끝에 프로필 필드 초기화 코드를 추가한다 (~line 1636)**

```javascript
      // 프로필 필드 초기화
      const profile = cachedProfile || { appName: '', nickname: '' };
      const appNameInput = document.getElementById('input-app-name');
      const nicknameInput = document.getElementById('input-nickname');
      if (appNameInput) appNameInput.value = profile.appName;
      if (nicknameInput) nicknameInput.value = profile.nickname;
```

- [ ] **Step 5: `renderSettings()` 함수 바로 뒤에 저장 핸들러 2개를 추가한다**

```javascript
    async function onSaveAppName() {
      const val = (document.getElementById('input-app-name').value || '').trim();
      try {
        await saveAppName(val);
        document.getElementById('mh-title-text').textContent = val || '루틴 지킴이';
      } catch {
        alert('저장에 실패했어요. 다시 시도해주세요.');
      }
    }

    async function onSaveNickname() {
      const val = (document.getElementById('input-nickname').value || '').trim();
      try {
        await saveNickname(val);
        document.getElementById('user-display-name').textContent = val || '나의 홈';
      } catch {
        alert('저장에 실패했어요. 다시 시도해주세요.');
      }
    }
```

- [ ] **Step 6: 헤더 앱 이름 span에 id를 부여한다 (~line 775)**

기존:
```html
          <div class="mh-title">루틴 지킴이</div>
```

변경:
```html
          <div class="mh-title" id="mh-title-text">루틴 지킴이</div>
```

- [ ] **Step 7: `renderMain()` 에서 `user-display-name` 갱신 코드를 수정한다 (~line 1259)**

기존:
```javascript
      if (currentUser && currentUser.displayName) {
        document.getElementById('user-display-name').textContent = currentUser.displayName + '의 홈';
      }
```

변경:
```javascript
      const profile = cachedProfile || { appName: '', nickname: '' };
      document.getElementById('mh-title-text').textContent = profile.appName || '루틴 지킴이';
      document.getElementById('user-display-name').textContent = profile.nickname || '나의 홈';
```

- [ ] **Step 8: `onAuthStateChanged` 콜백에서 `migrateFromLocalStorage()` 호출 뒤에 `loadUserProfile()` 호출을 추가한다 (~line 1887)**

기존:
```javascript
        await migrateFromLocalStorage();
        await initRoutineSettings();
        showScreen('main');
```

변경:
```javascript
        await migrateFromLocalStorage();
        await Promise.all([initRoutineSettings(), loadUserProfile()]);
        showScreen('main');
```

- [ ] **Step 9: 브라우저에서 수동 검증**

1. 설정 탭으로 이동
2. 프로필 섹션이 색상 꾸미기 위에 보이는지 확인
3. 앱 이름 입력란에 "수조의 일상" 입력 → 저장 → 헤더 타이틀이 "수조의 일상"으로 바뀌는지 확인
4. 닉네임 입력란에 "수조리카의 홈" 입력 → 저장 → 헤더 서브타이틀이 "수조리카의 홈"으로 바뀌는지 확인
5. 페이지 새로고침 후 로그인 → 저장한 이름/닉네임이 유지되는지 확인

- [ ] **Step 10: 커밋**

```bash
git add index.html
git commit -m "feat: 설정 탭에서 앱 이름 및 닉네임 변경 기능 추가"
```
