# Firebase 동기화 — 설계 문서

**작성일:** 2026-06-06  
**상태:** 승인됨  
**관련 스펙:** 2026-06-05-routine-keeper-design.md

---

## 개요

기존 localStorage 기반 저장 방식을 Firebase Firestore + Google 로그인으로 교체.  
같은 구글 계정으로 로그인하면 폰·컴퓨터 어디서든 동일한 스케줄·루틴 데이터를 볼 수 있다.

---

## 변경 범위

`index.html` 단일 파일만 수정. 추가 파일 없음.

---

## 로그인 UI

### 비로그인 상태
- 앱을 열면 기존 메인/스케줄 화면 대신 로그인 화면 표시
- "Google로 로그인" 버튼 하나만 노출
- 로그인 완료 시 메인 화면으로 전환

### 로그인 상태
- 기존 메인 화면 그대로 유지
- 헤더에 **로그아웃** 버튼 추가

### 로그인 화면 HTML (신규 추가)
```html
<div id="screen-login">
  <div class="login-card">
    <h1>루틴 지킴이</h1>
    <p>로그인하면 폰·컴퓨터에서 같은 데이터를 볼 수 있어요</p>
    <button onclick="signInWithGoogle()">Google로 로그인</button>
  </div>
</div>
```

---

## Firestore 데이터 구조

```
/users/{uid}/
  schedule/
    {dateKey}         →  "A" | "C" | "DO"   (예: "2026-06-05": "C")
  routines/
    {dateKey}         →  { routineName: boolean }
                         (예: "2026-06-05": { "환기": true, "청소기": false })
```

- `uid`: Firebase Google 로그인 후 발급되는 고유 사용자 ID
- `dateKey`: "YYYY-MM-DD" 형식

---

## 기존 localStorage 데이터 마이그레이션

첫 로그인 시 1회 실행:
1. localStorage의 `schedule`, `routines` 키 읽기
2. 데이터가 있으면 Firestore에 일괄 저장
3. localStorage 해당 키 삭제
4. 이후부터는 Firestore만 사용

---

## 데이터 읽기/쓰기 방식

| 기존 | 변경 후 |
|------|---------|
| `saveShift(dateKey, shift)` → localStorage | Firestore `/users/{uid}/schedule/{dateKey}` set |
| `loadShift(dateKey)` → localStorage | Firestore `/users/{uid}/schedule/{dateKey}` get |
| `saveRoutineCheck(dateKey, name, checked)` → localStorage | Firestore `/users/{uid}/routines/{dateKey}` update |
| `loadRoutineChecks(dateKey)` → localStorage | Firestore `/users/{uid}/routines/{dateKey}` get |

Firestore 읽기는 비동기(async/await). 기존 동기 함수들을 async로 변환.

---

## 오프라인 동작

Firebase SDK의 오프라인 퍼시스턴스(`enableIndexedDbPersistence`) 활성화.  
인터넷 없을 때도 캐시된 데이터로 앱 작동, 연결 복구 시 자동 동기화.

---

## Firebase 설정 (사용자가 직접 수행)

구현 전 사용자가 Firebase Console에서 직접 해야 하는 작업:

1. Firebase 프로젝트 생성
2. Firestore Database 활성화 (테스트 모드로 시작)
3. Authentication → Google 로그인 제공업체 활성화
4. 웹 앱 등록 → `firebaseConfig` 객체 복사
5. Firestore 보안 규칙 설정:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{uid}/{document=**} {
         allow read, write: if request.auth != null && request.auth.uid == uid;
       }
     }
   }
   ```

---

## 범위 밖

- 여러 사용자 지원 (앱은 항상 1인용)
- 실시간 리스너 (페이지 새로고침으로 충분)
- 오프라인 시 충돌 해결 전략 (단일 사용자라 충돌 없음)
