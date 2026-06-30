# `obj.toString()` 과 프로토타입 체인

### 예제

```js
const obj = {
  name: "javascript",
};

console.log(obj.toString()); // "[Object object]" ;
```

`obj` 에는 `name` 프로퍼티만 직접 정의되어 있다.

```js
const obj = {
  name: "javaScript",
};
```

그런데도 `obj.toString()`을 호출할 수 있다.

그 이유는 JavaScript 객체가 자신의 프로토타입과 연결되어 있기 때문이다.

> **프로토타입이란?**
>
> 객체가 다른 객체의 프로퍼티와 메소드를 상속받아, 사용할 수 있도록 연결된 객체

## `obj` 가 직접 가진 프로퍼티

객체가 직접 가지고 있는 프로퍼티를 **자체 프로퍼티** 라고 한다.

```js
const obj = {
  name: "javascript",
};
```

`obj` 가 직접 가진 프로퍼티는 `name` 이다.

```js
obj.hasOwnProperty("name");
// true

obj.hasOwnProperty("toString");
// false
```

`toString` 은 `obj`가 직접 가지고 있지 않다.
그런데도 다음 코드는 정상적으로 실행된다.

`ojb.toString()`

JavaScript 가 `obj` 에서 `toString`을 찾기 못하면 **프로토타입체인**을 따라 상위 객체에서 찾아내기 때문이다.

---

## 프로토타입이란?

프로토타입은 **객체가 다른 객체의 프로퍼티와 메서드를 사용할 수 있도록 연결된 객체**이다.

일반적인 객체 리터럴은 기본적으로 `Object.property` 과 연결된다.

> javaScript는 프로토타입을 기반으로 객체지향의 상속 개념을 구현한다.

```js
const obj = {
  name: "javascript",
};

Object.getPrototypeOf(obj) === Object.prototype; // true
```

구조를 표현하면 다음과 같다.

```
obj
├─ name: "javascript"
└─ [[Prototype]]
      ↓
Object.prototype
├─ toString()
├─ valueOf()
├─ hasOwnProperty()
└─ ...
```

`obj`의 내부 슬롯인 `[[Property]]` 이 `Object.prototype`을 가리킨다.

## 프로토타입 체인이란?

**상위 프로토타입과 연쇄적으로 연결된 구조**를 의미한다.

그리고 프로퍼티나 메서드에 접근하기 위해 이 연결 구조를 따라 차례대로 검색하는 것을 프로토타입 체이닝이라고 한다.

```
obj
 ↓
Object.prototype
 ↓
null

일반 객체의 프로토타입 체인은 보통 다음과 같다.

obj
  └─ [[Prototype]] → Object.prototype
                          └─ [[Prototype]] → null
```

`null`은 프로토타입 체인의 끝이다.

## `obj.toString()`을 찾는 과정

이 코드를 실행하면 javaScript는 다음 순서로 `toString`을 찾는다.

### 1단계: `ojb` 자신에게서 찾는다.

```
obj
├─ name
└─ toString 없음
```

obj에는 toString이 없다.

### 2단계: 프로토타입으로 이동핳ㄴ다.

```
obj.[[Prototype]]
        ↓
Object.prototype
```

### 3단계: `Object.prototype` 에서 찾는다.

```
Object.prototype
└─ toString 있음
```

`Object.prototype`에서 `toString` 메서드를 발견한다.

### 4단계: 찾은 메서드를 실행한다.

결과적으로는 다음 메서드가 실행된다.

```js
Object.prototype.toString.call(obj);
```

따라서 다음 결과가 나온다.

```js
"[object Object]";
```

## 왜 `[object Object]` 가 나오는가?

`Object.prototype.toString`은 일반적으로 다음 형식의 문자열을 반환한다.

```
[object 타입 이름]
```

`obj`는 일반 객체이므로 타입 이름이 `Object` 가 된다.

즉, `[object Object]` 는 **이 값은 일반 Object 객체이다.**

객체 내부의 프로퍼티 값을 보여주는 문자열이 아니다.

### `"javascript"` 가 나오지 않는 이유

다음 코드에서 `"javascript"` 는 `obj` 전체가 아니라, `name` 프로퍼티의 값이다.

```js
const obj = {
  name: "javascript",
};
```

따라서 `"javascript"` 를 얻으려면 `name` 프로퍼티에 접근해야 한다.

`obj.name;` 이렇게하면 얻을 수 있다. 또는 문자열 값에 `toString()`을 호출하는 방법도 있다.

```js
obj.name.toString();
// "javascript"
```

호출 대상을 비교하면 다음과 같다.

```js
obj.name.toString();
// name 프로퍼티의 값인 문자열을 문자열로 반환
// "javascript"

obj.toString();
// obj 객체 전체를 기본 문자열로 변환
// "[object Object]"
```

`toString()`은 객체 안에서 문자열 값을 자동으로 찾아주는 메서드가 아니다.

호출한 값 자체를 문자열로 표현하는 메서드이다.

## 자체 프로퍼티가 프로토타입보다 우선한다.

JavaScript는 항상 객체 자신에서 먼저 프로퍼티를 찾는다.

따라서 `obj`에 직접 `toString`을 정의하면 해당 메서드가 먼저 사용된다.

```js
const obj = {
  name: "javascript",

  toString() {
    return this.name;
  },
};
console.log(obj.toString());
// "javascript";
```

객체 자신에게서 `toString`을 찾았기 때문에 `Object.prototype`까지 올라가지 않는다.

이를 **프로퍼티 섀도잉** 이라고 한다.

> 객체 자신의 프로퍼티가 프로토타입과 같은 이름을 가진 프로퍼티를 가리는 현상이다.

### 프로퍼티 섀도잉 예시

```js
const obj = {
  toString() {
    return "직접 정의한 메서드";
  },
};

obj.toString();
// "직접 정읳나 메서드"
```

두 개의 `toString` 이 존재하는 구조이다.

두 개의 toString이 존재하는 구조다.

```
obj
└─ toString()                ← 먼저 발견됨
     ↓

Object.prototype
└─ toString()                ← 사용되지 않음
```

프로토타입의 메서드가 사라진 것은 아니다.

객체 자신의 메서드가 먼저 발견되어 가려진 것이다.

### 외울 문장

- 객체에서 프로퍼티나 메서드를 찾을 때 객체 자신에게 없으면 프로토타입 체인을 따라 상위 객체에서 찾는다.
- `obj.toString()`은 `obj`가 직접 가진 메서드가 아니라 `Object.prototype`에서 상속받은 메서드이다.
- 일반 객체가 `Object.prototype.toString()`을 호출하면 기본 문자열 표현인 `[object Object]`가 반환된다.
