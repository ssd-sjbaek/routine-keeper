# Drag-to-Reorder Routines Design Spec

## Overview

루틴 목록에 드래그로 순서를 바꿀 수 있는 기능을 추가한다. 편집 모드에서만 활성화되어 일반 체크 모드에서 실수로 순서가 바뀌는 일을 방지한다.

## Architecture

- 라이브러리: Sortable.js 1.15.2 (CDN)
- 드래그 가능 조건: `routineEditMode === true`일 때만
- 저장 방식: 드래그 완료 시 DOM 순서를 기준으로 `currentRoutines` 배열 업데이트 → `saveRoutineList()` 호출
- 새 Firestore 컬렉션 불필요 — 기존 `routineList` 저장 구조 그대로 사용

## UI Changes

### 편집 모드 루틴 아이템 구조
```
[≡]  루틴 이름  [✕]
```
- `≡` : 드래그 핸들 (`.drag-handle` 클래스), `cursor: grab`
- 핸들을 잡아야만 드래그 시작 (Sortable `handle` 옵션)
- `✕` : 기존 삭제 버튼 유지

### 일반 모드 (체크 모드)
변화 없음 — `≡` 핸들 없음, Sortable 초기화 없음

### 드래그 중 시각 피드백
Sortable.js 기본 ghost 효과 사용 (반투명 복사본)

## Implementation Details

### CDN 추가
```html
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.2/Sortable.min.js"></script>
```
기존 Firebase CDN 스크립트 아래에 추가.

### Sortable 초기화 (renderMain 편집 모드 분기)
```javascript
const sortable = Sortable.create(routineListEl, {
  handle: '.drag-handle',
  animation: 150,
  onEnd() {
    const newOrder = Array.from(routineListEl.querySelectorAll('[data-routine]'))
      .map(el => el.dataset.routine);
    currentRoutines = newOrder;
    saveRoutineList(currentRoutines);
  }
});
```

### 편집 모드 루틴 아이템 HTML
```html
<div class="routine-edit-item" data-routine="루틴이름">
  <span class="drag-handle">≡</span>
  <span class="routine-edit-name">루틴이름</span>
  <button onclick="deleteRoutine('루틴이름')">✕</button>
</div>
```

### CSS
```css
.drag-handle {
  cursor: grab;
  padding: 0 8px;
  color: #aaa;
  user-select: none;
}
.drag-handle:active {
  cursor: grabbing;
}
```

## Constraints

- 편집 모드 전용: `routineEditMode === true`일 때만 Sortable 초기화
- 일반 모드 렌더링에는 핸들 및 Sortable 없음
- 순서 저장은 기존 `saveRoutineList(array)` 함수 재사용
- `currentRoutines` 배열이 순서의 단일 소스

## Out of Scope

- 투두 아이템 드래그 (이번 스펙 미포함)
- 루틴과 투두 간 드래그
- 애니메이션 커스터마이징
