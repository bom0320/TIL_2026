# Biome (lint) 사용 정리

## 1. 실행 명령어

```bash
pnpm lint:fix
# 또는
npx biome check --write .
```

---

## 2. 이 명령어가 하는 일

`biome check --write .`는 다음을 동시에 수행한다:

### 1) 코드 검사 (Lint)

- `any` 타입 사용
- 사용되지 않는 변수
- 위험한 코드 패턴 (ex. document.cookie)

### 2) 코드 포맷팅 (Format)

- 들여쓰기 정리
- import 정렬
- 세미콜론 / trailing comma 정리
- 줄바꿈 정리

### 3) 자동 수정

- 수정 가능한 항목은 자동으로 코드 반영

---

## 3. 왜 수시로 실행해야 하는가

Biome는 단순 포맷터가 아니라 **코드 상태를 정리하는 도구**이기 때문에
실행하지 않으면 변경사항이 누적된다.

### 실행하지 않을 경우 문제

- PR diff가 불필요하게 커짐
- import / formatting 변경이 로직 변경과 섞임
- 리뷰 가독성 저하
- CI에서 lint 에러 발생 가능

---

## 4. 권장 사용 타이밍

### 필수

- PR 생성 전

### 권장

- 작업 단위 완료 후

```bash
작업 → lint:fix → 커밋 → PR
```

---

## 5. 커밋 전략

Biome 적용은 별도 커밋으로 분리하는 것이 좋다.

```bash
chore: apply biome lint fixes
```

이유:

- 기능 변경과 스타일 변경 분리
- 리뷰 시 diff 파악 용이

---

## 6. 자동 수정 vs 수동 수정

### 자동 수정됨

- import 정렬
- 코드 포맷
- 일부 unused 코드

### 수동 수정 필요

- any 타입 경고
- 로직 관련 문제
- 일부 lint 규칙 (ex. document.cookie)

---

## 7. 주의사항

- 실행 후 반드시 diff 확인
- import 경로(alias) 깨짐 여부 확인
- 자동 삭제된 변수 확인

---

## 8. 결론

Biome는 한 번에 돌리는 도구가 아니라
**작업 흐름에 포함시켜야 하는 필수 도구**

- "PR 전에 한 번"이 아니라
- "작업 단위마다 실행"이 정상적인 사용 방식
