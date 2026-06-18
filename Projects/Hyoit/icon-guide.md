# 효잇 모바일 아이콘 사용 가이드

## 1. 기본 원칙

효잇 모바일 앱에서는 아이콘을 직접 `@expo/vector-icons`에서 가져다 쓰지 않고, 프로젝트에서 정의한 `IconSymbol` 컴포넌트를 통해 사용한다.

```tsx
// 권장
import { IconSymbol } from "@/src/parent/shared/ui/IconSymbol";

<IconSymbol name="bell.fill" size={28} color="#555555" />;
```

```tsx
// 비권장
import MaterialIcons from "@expo/vector-icons/MaterialIcons";

<MaterialIcons name="notifications-none" size={28} color="#555555" />;
```

아이콘 사용을 `IconSymbol`로 통일하면 다음 장점이 있다.

- 아이콘 라이브러리 의존성을 한 곳에서 관리할 수 있다.
- iOS의 SF Symbol 이름과 Android/Web의 Material Icons 이름을 매핑해서 사용할 수 있다.
- 화면마다 다른 아이콘 네이밍을 쓰는 문제를 줄일 수 있다.
- 이후 아이콘 교체나 디자인 시스템 정리가 쉬워진다.

---

## 2. 현재 아이콘 구조

현재 효잇 모바일 앱은 다음 파일에서 아이콘 매핑을 관리한다.

```txt
apps/mobile/src/parent/shared/ui/IconSymbol.tsx
```

`IconSymbol`은 내부적으로 `MaterialIcons`를 사용한다.

```ts
import MaterialIcons from "@expo/vector-icons/MaterialIcons";
```

하지만 실제 컴포넌트 사용 시에는 `MaterialIcons`의 이름을 직접 넘기지 않는다.
대신 `MAPPING`에 등록된 이름만 사용한다.

---

## 3. 아이콘 추가 방법

새 아이콘이 필요하면 먼저 `IconSymbol.tsx`의 `MAPPING`에 등록한다.

예시:

```ts
const MAPPING = {
  "house.fill": "home",
  "gamecontroller.fill": "sports-esports",
  "text.bubble.fill": "chat-bubble",
  "person.fill": "person",
  "paperplane.fill": "send",
  "chevron.left.forwardslash.chevron.right": "code",
  "chevron.right": "chevron-right",
  "bell.fill": "notifications-none",
  pencil: "edit",
} as const satisfies IconMapping;
```

왼쪽 key는 앱 코드에서 사용할 이름이다.

```ts
"bell.fill";
```

오른쪽 value는 실제 `MaterialIcons` 아이콘 이름이다.

```ts
"notifications-none";
```

사용할 때는 왼쪽 key를 사용한다.

```tsx
<IconSymbol name="bell.fill" size={28} color="#555555" />
```

---

## 4. 아이콘 이름 정하는 기준

아이콘 key는 가능하면 SF Symbols 스타일에 맞춘다.

예시:

| 의미          | IconSymbol name       | MaterialIcons name   |
| ------------- | --------------------- | -------------------- |
| 홈            | `house.fill`          | `home`               |
| 게임          | `gamecontroller.fill` | `sports-esports`     |
| 대화          | `text.bubble.fill`    | `chat-bubble`        |
| 사람          | `person.fill`         | `person`             |
| 보내기        | `paperplane.fill`     | `send`               |
| 오른쪽 화살표 | `chevron.right`       | `chevron-right`      |
| 알림          | `bell.fill`           | `notifications-none` |
| 수정          | `pencil`              | `edit`               |

새 아이콘을 추가할 때는 기존 네이밍 흐름을 따른다.

```ts
"bell.fill";
"person.fill";
"paperplane.fill";
```

단, SF Symbol과 정확히 1:1로 맞추기 어렵다면 의미가 명확한 이름을 사용한다.

---

## 5. 사용 예시

### 알림 아이콘

```tsx
import { IconSymbol } from "@/src/parent/shared/ui/IconSymbol";

<IconSymbol name="bell.fill" size={28} color="#555555" />;
```

### 오른쪽 화살표

```tsx
<IconSymbol name="chevron.right" size={24} color="#777777" />
```

### 탭바 아이콘

```tsx
<IconSymbol name="text.bubble.fill" size={24} color={color} />
```

---

## 6. 금지 사항

아래 방식은 지양한다.

### 1. 화면에서 직접 MaterialIcons import

```tsx
import MaterialIcons from "@expo/vector-icons/MaterialIcons";

<MaterialIcons name="notifications-none" size={28} color="#555555" />;
```

아이콘 관리 지점이 분산되므로 사용하지 않는다.

---

### 2. 등록되지 않은 name 사용

```tsx
<IconSymbol name="notifications-none" size={28} color="#555555" />
```

`notifications-none`은 MaterialIcons 이름이다.
`IconSymbol`에서는 `MAPPING`에 등록된 key를 사용해야 한다.

올바른 예시는 다음과 같다.

```tsx
<IconSymbol name="bell.fill" size={28} color="#555555" />
```

---

### 3. 이모지로 아이콘 대체

```tsx
<Text>🔔</Text>
```

임시 구현에서는 사용할 수 있지만, 실제 UI 구현에서는 `IconSymbol`을 사용한다.

---

## 7. 새 아이콘 추가 절차

새 아이콘이 필요할 때는 아래 순서로 작업한다.

1. 필요한 아이콘 의미를 정한다.
2. `MaterialIcons`에서 사용할 실제 아이콘 이름을 찾는다.
3. `IconSymbol.tsx`의 `MAPPING`에 key-value를 추가한다.
4. 화면에서는 `IconSymbol`의 key를 사용한다.
5. 아이콘 크기와 색상은 사용하는 화면에서 props로 조정한다.

예시:

```ts
const MAPPING = {
  ...
  "bell.fill": "notifications-none",
} as const satisfies IconMapping;
```

```tsx
<IconSymbol name="bell.fill" size={28} color="#555555" />
```

---

## 8. 타입 기준

`IconSymbol`의 `name`은 `MAPPING`의 key만 허용한다.

```ts
type IconSymbolName = keyof typeof MAPPING;
```

따라서 `MAPPING`에 등록되지 않은 아이콘 이름은 사용할 수 없다.
이 방식은 오타를 줄이고, 프로젝트에서 허용한 아이콘만 사용하게 만든다.

---

## 9. 권장 컴포넌트 형태

현재 `IconSymbol.tsx`는 아래 구조를 기준으로 유지한다.

```tsx
import MaterialIcons from "@expo/vector-icons/MaterialIcons";
import { type SymbolWeight } from "expo-symbols";
import { ComponentProps } from "react";
import {
  type OpaqueColorValue,
  type StyleProp,
  type TextStyle,
} from "react-native";

type IconMapping = Record<string, ComponentProps<typeof MaterialIcons>["name"]>;
type IconSymbolName = keyof typeof MAPPING;

const MAPPING = {
  "house.fill": "home",
  "gamecontroller.fill": "sports-esports",
  "text.bubble.fill": "chat-bubble",
  "person.fill": "person",
  "paperplane.fill": "send",
  "chevron.left.forwardslash.chevron.right": "code",
  "chevron.right": "chevron-right",
  "bell.fill": "notifications-none",
  pencil: "edit",
} as const satisfies IconMapping;

export function IconSymbol({
  name,
  size = 24,
  color,
  style,
}: {
  name: IconSymbolName;
  size?: number;
  color: string | OpaqueColorValue;
  style?: StyleProp<TextStyle>;
  weight?: SymbolWeight;
}) {
  return (
    <MaterialIcons
      name={MAPPING[name]}
      size={size}
      color={color}
      style={[
        {
          includeFontPadding: false,
          lineHeight: size + 2,
          textAlign: "center",
        },
        style,
      ]}
    />
  );
}
```

---

## 10. 정리

효잇 모바일 앱에서 아이콘을 사용할 때는 다음 기준을 따른다.

```txt
아이콘 직접 import 금지
IconSymbol 사용
새 아이콘은 MAPPING에 먼저 등록
화면에서는 MAPPING key만 사용
이모지 아이콘은 실제 UI에서 사용하지 않기
```

예시:

```tsx
<IconSymbol name="bell.fill" size={28} color="#555555" />
```
