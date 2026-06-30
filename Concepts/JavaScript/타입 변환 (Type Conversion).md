# 타입 변환 (Type Conversion)

## 한줄 정의

자바스크립트의 타입 변환은 값이 연산자나 함수가 기대한는 타입에 맞게 다른 타입으로 바뀌는 과정이다.

### 1. 동적 타입 언어

자바스크립트는 변수의 타입이 아니라 **할당된 값의 타입** 에 따라 타입이 결정된다.

```js
let dynamicValue = 7; // Number 타입
dynamicValue = "Hello, world!"; // String 타입
dynamicValue = true; // Boolean 타입
```

즉, 하나의 변수에 처음 할당된 값의 타입과 상관없이 다른 타입의 값을 다시 할당할 수 있다.

## 2. 암시적 타입 변환

암시적 타입 변환은 개발자가 직접 변환하지 않아도 자바스크립트가 상황에 맞게 자동으로 타입을 바꾸는 것이다.

타입 강제 변환에서 피연산자가 어떤 타입으로 변환될지는 **연산자에 따라 달라진다.**

예를 들어 `+` 연산자는 숫자 덧셈도 하지만 문자열 연결도 할 수 있다. 따라서 한쪽 피연산자가 문자열이면 다른 피연산자도 문자열 타입으로 변환을 시도한다.

```js
7 + "13"; // "713"
```

객체 타입이 피연산자로 오면, 먼저 객체를 원시 타입으로 변환한 뒤 연산을 수행한다.

```js
const value = {
  toString() {
    return "13";
  },
};

7 + value; // "713"
```

반면 `-` 연산자는 문자열 연결 기능이 없고 숫자 연산만 처리한다.
따라서 피연산자를 숫자 타입으로 변환하려고 한다.

```js
7 - "3"; // 4
```

단, `BigInt`가 포함된 경우에는 주의해야한다.
`-` 연산자는 피연산자가 모두 `BigInt` 타입이면 `BigInt` 연산을 수행하지만, 그렇지 않으면 `Number` 타입으로 변환을 시도한다.

```js
10n - 3n; // 7n
10 - "3"; // 7
```

하지만 `BigInt` 는 `Number`를 섞어서 연산하면 에러가 발생한다.

```js
10n - 3; // TypeError
```

## 3. 명시적 타입 변환

명시적 타입 변환은 개발자가 직접 타입을 바꾸는 것이다.

대표적으로 다음 함수를 사용한다.

```js
Number(value);
String(value);
Boolean(value);
Object(value);
```

## 4. Number 타입 변환

Number 타입으로 변환될 때는 값의 타입에 따라 규칙이 다르다.

| 타입      | 변환 규칙                                          |
| --------- | -------------------------------------------------- |
| Null      | `0`으로 변환                                       |
| Undefined | `NaN`으로 변환                                     |
| Boolean   | `true`는 `1`, `false`는 `0`으로 변환               |
| String    | Number 리터럴로 파싱되면 숫자로 변환, 아니면 `NaN` |
| BigInt    | 암시적 변환은 타입 에러                            |
| Symbol    | 타입 에러                                          |

```js
Number(null); // 0
Number(undefined); // NaN
Number(true); // 1
Number(false); // 0
Number("123"); // 123
Number("hello"); // NaN
```

## 5. Object 타입이 Number 타입으로 변환되는 과정

Object 타입은 원시 타입으로 먼저 변환된 뒤 Number 타입으로 변환된다.

객체를 원시값으로 바꿀때는 다음 메서드가 사용된다.

- Symbol.toPrimitive
- valueOf
- toString

### 5.1 Symbol.toPrimitive

`Symbol.toPrimitive` 메서드는 **객체를 원시값으로 변환할 때 가장 먼저 호출된다.**
이때 `hint` 값이 전달된다.

```
number
string
default
```

ex)

```js
const luckyNumber = {
  [Symbol.toPrimitive](hint) {
    switch (hint) {
      case "number":
        return 7;
      case "string":
        return "칠";
      case "default":
        return "seven";
    }
  },
};

console.log(+luckyNumber); // 7
console.log(`${luckyNumber}`); // "칠"
console.log("" + luckyNumber); // "seven"
```

- `+luckyNumber`는 Number타입을 기대하므로 `hint`가 `"number"` 로 전달된다.
- 템플릿 리터럴은 String 타입을 기대하므로 `hint`가 `"string"`으로 전달된다.
- 문자열 연결처럼 숫자와 문자열 둘 다 가능할 때는, `"default"` 가 전달된다.

### 5.2 valueOf

`valueOf` 메서드는 기본적으로 객체 자기 자신을 반환한다.

하지만 객체 안에서 직접 정의하면 원하는 원시값을 반환하게 만들 수 있다. 타입 힌트를 인자로 받는 Symbol.toPrimitive 메서드와 달리 아무 인자도 받지 않으며 반환값이 꼭 원시값일 필요도 없다.

```js
const luckNumber = {
  valueOf() {
    return 7;
  },
};

console.log(+luckyNumber); // 7
```

### 5.3 toString

`toString` 메서드는 객체를 문자열로 변환하기 위한 메서드이다.

```js
const luckyNumber = {
  toString() {
    return "칠";
  },
};

console.log(String(luckyNumber)); // "칠"
```

객체가 Number 타입으로 변환될 때는 보통 다음 순서로 원시값을 찾는다.

```
Symbol.toPrimitive("number")
→ valueOf()
→ toString()
```

## 6. 단항 `+`연산자

단항 `+` 연산자는 값을 Number 타입으로 강제 변환하는 대표적인 연산자이다.

```js
console.log(+"123"); // 123
console.log(+true); // 1
console.log(+false); // 0
```

단, BigInt 타입에는 예외적으로 타입 에러가 발생한다.

```js
console.log(-123n); // -123n
console.log(+123n); // TypeError
```

## 7. 비트 연산과 TypedArray

자바스크립트의 숫자 타입은 `Number` 과 `BigInt`이다.

하지만 실제 이진 데이터를 저장하기 위해 고정 너비 정수 타입도 사용된다.

대표적으로는 `TypedArray`가 있다.

비트 연산은 Number 타입 값을 32비트 정수로 변환해서 처리한다.
따라서 32 비트 버무이를 넘는 값은 손실될 수 있다.

```js
0x100000000 >> 1; // 0
0x100000000n >> 1n; // 2147483648n
```

Number 타입 비트 연산은 32비트 정수로 변환되기 때문에 큰 수에서는 기대와 다른 결과가 나올 수 있다.
반면 BigInt는 임의 정밀도 정수이기 때문에 값이 보존된다.

## 8. String 타입 변환

String 타입으로 변환될 때는 값의 타입에 따라 다음 규칙이 적용된다.

| 타입      | 변환 규칙                                     |
| --------- | --------------------------------------------- |
| Undefined | `"undefined"`로 변환                          |
| Null      | `"null"`로 변환                               |
| Boolean   | `true`는 `"true"`, `false`는 `"false"`로 변환 |
| Number    | 10진수 형식의 문자열로 변환                   |
| BigInt    | 10진수 형식의 문자열로 변환                   |
| Symbol    | 타입 에러                                     |

```js
String(undefined); // "undefined"
String(null); // "null"
String(true); // "true"
String(123); // "123"
String(123n); // "123n"
```

## 9. Object 타입이 String 타입으로 변환되는 과정

Object 타입을 String 타입으로 변환할 때도 객체는 먼저 원시값으로 변환된다.

Number 타입 변환과 차이점은 호출 순서이다.

```
Symbol.toPrimitive("string")
→ toString()
→ valueOf()
```

즉, String 타입 변환에서는 `toString()` 이 `valueOf()` 보다 먼저 사용된다.

## 10. 문자열 연결

String 타입으로 강제 변환을 일으키는 대표적인 연산은 문자열 연결이다.

문자열 연결 방법에는 다음이 있다.

```js
"행운의 숫자: " + luckyNumber;
"행운의 숫자: ".concat(luckyNumber);
`행운의 숫자: ${luckyNumber}`;
```

- **`+` 연산자 :** `Symbol.toPrimitive("default")` -> `valueOf()` -> `toString()`

- **`concat`, 템플릿 리터럴 :** `Symbol.toPrimitive("default")` -> `toString()` -> `valueOf()`

ex)

```js
const luckyNumber = {
  valueOf() {
    return 7;
  },
  toString() {
    return "칠";
  },
};

console.log("행운의 숫자: " + luckyNumber);
// "행운의 숫자: 7"

console.log(`행운의 숫자: ${luckyNumber}`);
// "행운의 숫자: 칠"
```

## 11. Boolean 타입 변환

Boolean 타입으로 변환될 때는 다음 값들이 `false` 가 된다.

| 타입      | false로 변환되는 값 |
| --------- | ------------------- |
| Undefined | `undefined`         |
| Null      | `null`              |
| Number    | `0`, `-0`, `NaN`    |
| BigInt    | `0n`                |
| String    | 빈 문자열 `""`      |
| Symbol    | 없음                |
| Object    | 없음                |

나머지 값들은 `true` 로 변환된다.

```js
console.log(!!undefined); // false
console.log(!!7); // true
console.log(Boolean("false")); // true -> Boolean으로 변환할 때 문자열은 "무슨 내용인지"는 보지 않고, "비어 잇는지"만 본다.
console.log(!!{}); // true
console.log(!![]); // true
```

빈 객체와 빈 배열도 객체이기 때문에 `true`로 평가된다.

## 12. 객체(Object) 타입 변환

다른 원시 타입을 Object 타입으로 변환할 수도 있다.

이때 자바스크립트 래퍼 객체를 생성한다.

```js
const seven = Object.prototype.valueOf.call(7);

console.log(seven); // Number {7}
console.log(typeof seven); // object
console.log(seven.valueOf()); // 7
console.log(seven + 3); // 10
```

`Object()` 함수도 원시값을 객체로 변환한다.

```js
Object(null); // {}
Object(undefined); // {}
```

단, `Object.prototype.valueOf.call(null)`처럼 `null` 이나, `undefined`에 대해 직접 객체 변환을 시도하면 에러가 발생한다.
