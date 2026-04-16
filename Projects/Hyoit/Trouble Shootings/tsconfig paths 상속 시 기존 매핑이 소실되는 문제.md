# [Troubleshooting] tsconfig paths 상속 시 기존 매핑이 소실되는 문제

## 문제 현상

모노레포(Monorepo) 환경에서 루트의 `tsconfig.base.json` 을 상속받아 개별 프로젝트(예: mobile)의 tsconfig.json 을 설정하던 중, 특정 패키지(예: `@hyoit/auth` ) 를 찾지 못하는 모듈 참조 에러가 발생함

### 에러 메시지:

> `Cannot find module '@hyoit/auth' or its corresponding type declarations.`

### 상황:

루트 설정에는 `@hyoit/*` 경로가 정의되어 있으나, 하위 프로젝트에서 `@/*` 경로를 추가하마자 발생함

---

## 원인 분석

TypeScript의 `extends` 동작 방식 때문이다. `tsconfig` 에서 **paths 필드는 ‘Deep Merge’가 아니라, ‘Override’ 방식으로 동작**한다.

> “하나를 수정하려고 이름을 부르는 순간, 그 폴더(객체) 안에 들어있던 기존 내용물은 싹 다 지워지고 새 내용으로 교체된다”

1. **부모(`tsconfig.base.json` ) :** `@hyoit/auth`, `@hyoit/storage` 등 공통 경로 정의
2. **자식(`apps/mobile/tsconfig.json` ) :** 앱 내부 경로인 `@/*` 정의
3. **결과 :** 자식에서 `paths` 객체를 선언하는 순간, 부모의 `paths` 객체 전체가 자식의 객체로 **통째로 교체**된다. 부모에만 존재하던 설정들은 모두 무시된다.

---

## 해결 방법

### 방법 1: 자식 설정에서 부모 경로 재정의 (가장 확실한 방법)

가장 직관적인 해결책은 자식 프로젝트의 paths 에 부모의 경로 설정을 다시 포함시키는 것이다.

```json
// apps/mobile/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      // 1. 부모의 paths를 수동으로 다시 명시 (상대 경로 주의)
      "@hyoit/auth": ["../../packages/auth/src"],
      "@hyoit/storage": ["../../packages/storage/src"],

      // 2. 현재 앱에서만 쓸 별칭 추가
      "@/*": ["./src/*"]
    }
  }
}
```

### 방법 2: 프로젝트 참조(Project References) 활용

모노레포에서 추천하는 방식이다. paths 에 의존하기 보다 실제 프로젝트 간의 관계를 설정한다.

이 방식을 쓰면 개별 패키지의 `package.json` 이름을 그대로 인식하므로 paths 오버라이드 문제에서 자유로워질 수 있다.

```json
// apps/mobile/tsconfig.json
{
  "compilerOptions": {
    "composite": true
  },
  "references": [
    { "path": "../../packages/auth" },
    { "path": "../../packages/storage" }
  ]
}
```

### 방법 3: 루트 설정을 더 범용적으로 유지

만약 앱 내부 별칭(`@/*` ) 이 모든 패키지에서 공통으로 쓰이는 구조라면, 아예 루트(`base.json` ) 단계에서 규칙을 정해두는 방법도 있다. (단, 프로젝트 구조가 정형화되어 있어야 한다)

---

## 요약 및 교훈

- tsconfig 의 compilerOptions 내 객체 형태 설정(특히 `paths`)은 **머지(Merge) 되지 않고 덮어씌워진다.**
- 하위 프로젝트에서 별칭을 추가할 때는 반드시 **기존 부모의 별칭이 누락되지 않았는지 점검**해야 한다.
- 모노레포 규모가 커진다면 `paths` 관리보다는, Project References나 pnpm workspace의 실제 의존성 해결 방식을 활용하는 것이 유지보수에 유리하다.
