# 루틴 이름 인라인 편집 스펙

## 개요

편집 모드에서 루틴 이름을 탭하면 인라인 `<input>`으로 바뀌어 이름을 수정할 수 있다.

---

## 1. UI 동작

- 편집 모드(`routineEditMode = true`)에서 각 루틴 항목의 이름 `<span class="routine-name">` 탭 → 같은 위치에 `<input>`으로 교체
- input은 현재 이름을 `value`로, `maxlength="50"`, `flex: 1` 너비로 표시
- 저장 트리거: **엔터 키** 또는 **포커스 아웃(blur)**
- 자동 포커스: input 교체 직후 `input.focus()` + `input.select()` 호출

## 2. 저장 로직

| 조건 | 처리 |
|------|------|
| 빈 값 | 저장 안 함, 원래 이름 span 복원 |
| 원래 이름과 동일 | 저장 안 함 (Firestore 불필요), 원래 이름 span 복원 |
| 중복 이름 (자신 제외) | 저장 안 함, 원래 이름 span 복원 |
| 유효한 이름 | `currentRoutines[index]` 업데이트 → `saveRoutineList()` 호출 → span 텍스트 업데이트 |

저장 성공 시 `<input>`을 다시 `<span class="routine-name">`으로 교체한다 (전체 재렌더 없이).

## 3. 구현 방식

`renderMain()` 편집 모드 항목 생성 코드에서 `<span class="routine-name">` 요소에 `onclick` 핸들러를 추가한다.

```
span.onclick → startRoutineEdit(span, index, currentName)
```

`startRoutineEdit(span, index, name)`:
1. span을 `<input>`으로 교체
2. blur / keydown(Enter) 이벤트 등록
3. `commitRoutineEdit(input, span, index, originalName)` 호출로 처리

`commitRoutineEdit(input, span, index, originalName)`:
1. 값 trim
2. 빈 값 또는 중복 → span 복원
3. 유효 → `currentRoutines[index]` 갱신, `saveRoutineList()`, span 텍스트 업데이트

## 4. 스코프 외

- 편집 모드 진입/종료 시 열려 있는 input 자동 커밋 (현재 '완료' 버튼 탭 시 재렌더가 일어나므로 자연스럽게 처리됨)
- 루틴 추가 시 이름 중복 검사 (기존 코드 변경 없음)
