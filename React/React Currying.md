# React Currying

## 커링(Currying)이란?

- 여러 개의 인자를 가진 함수를 호출할 때, 한 번에 모든 인자를 전달하는 대신, 한번에 하나의 인자만 받는 일련의 함수들로 분해하는 프로그래밍 기법이다.
- 쉽게 얘기하면 **함수안에 함수를 반환하는 함수**로 이해하면 된다.
- 장점: 커링을 통해 함수를 재사용하기 쉽게 만들 수 있다.
- 또한, 부분 적용(partial application)을 통해 인자 일부를 미리 적용한 새로운 함수를 생성할 수 있다.
  - 예를 들어, multiply 함수를 커링하여 특정 값을 고정시킨 함수를 만들 수 있다.
    주요 특징과 활용 사례는 다음과 같다.

```js
function multiply(a, b) {
  // 파라미터로 두 인자를 받아 곱셈을 반환하는 함수
  return a * b;
}

function curryingMul(a) {
  // 함수 안에 함수가 들어간 curryingMul 함수
  return function (b) {
    return multiply(a, b);
  };
}

const multiplyByTwo = curryingMul(2); // 출력 예제
console.log(multiplyByTwo(3)); // 출력: 6이라고 나옴
```

### 1. 기본 개념

일반적인 함수 `f(a, b)`를 `f(a)(b)` 형태의 호출로 변환한다.

```js
// 일반 함수
const add = (a,b) => a + b;

// 커링 함수
const curriedAd d= (a) => (b) => a + b;
curriedAdd(1)(2); // 3
```

### 2. React에서의 주요 활동: 이벤트 핸들러

React 컴포넌트에서 리스트를 렌더링할 때, 각 항목의 ID와 같은 특정 데이터를 이벤트 핸들러에 전달하기 위해 자주 사용된다.

- **커리 미사용(인라인 익명 함수) :** 매 렌더링마다 새로운 함수가 생성되어 성능에 영향을 줄 수 있다.

```js
<button onClick={() => handleClick(id)}>삭제</button>
```

- **커링 사용 :** 핸들러를 미리 정의하여 코드를 깔끔하게 관리할 수 있다.

```jsx
const handleDelete = (id) => (event) => {
  console.log(`${id}번 항목 삭제`);
};

<button onClick={handleDelete(item.id)}>삭제</button>;
```

### 3. 커링의 장점

- **재사용성 :** 첫 번째 인자(설정값 등)를 고정한 재사용 가능한 "설정 함수"를 만들 수 있다.
- **가독성 :** 매개변수가 많은 복잡한 로직을 단계별로 분리하여 코드의 의도를 명확히 할 수 있다.
- **함수 조합 :** 고차 컴포넌트(HOC)나 미드루에어 구성 시 함수를 유연하게 결합하기 용이하다.

---

## 고차 함수(Higher-order function)란?

- 고차함수(Higher-Order Function,HOF)란 함수를 다루는 함수로, 다음 두 가지의 조건 중 하나 이상을 만족하는 함수를 말한다.

1. 함수를 인자(매개변수)로 받는 함수
2. 함수를 결과로 반환하는 함수 (-> 이게 '커링'에 해당함)

- 즉, 쉽게 말하면 고차 함수는 다른 함수를 인자로 받거나 함수를 반환하는 함수를 의미한다고 볼 수 있다.

```js
const function1 = (func) => (word) => {
  func();
  console.log(`${word}`);
};

const sayHello = () => console.log("Hello");

const higherOrderFunction = function1(sayHello);
higherOrderFunction("World");
// 출력: Hello World로 나옴
```

### 1. 주요 예시(JavaScript)

자바스크립트에서 매일 사용하는 배열 메서드들이 대표적인 고차 함수이다.

- **`map()` :** 배열의 각 요소에 실행할 '함수'를 인자로 받음
- **`filter()` :** 조건을 체크하는 '함수'를 인자로 받아 특정 요소만 걸러낸다.
- **`reduce()` :** 누적 계산을 위한 '함수'를 인자로 받는다.

### 2. React에서의 활용: 고차 컴포넌트(HOC)

React에서는 이를 응용하여 고차 컴포넌트(Higher-Order-Component) 패턴을 사용한다.

- **정의 :** 컴포넌트를 인자로 받아, 새로운 기능을 추가한 '새 컴포넌트'를 반환하는 함수이다.
- **용도 :** 로그인 여부 체크(인증), 로딩 애니메이션 처리,로깅 등 여러 컴포넌트에서 공통으로 쓰이는 로직을 분리할 때 유용하다.

---

## HOC(고차 컴포넌트)

- React에서 HOC(고차 컴포넌트)는 **컴포넌트를 인자로 받아 새로운 컴포넌트를 반환하는 함수**를 의미한다.
- 주로 HOC 컴포넌트 간의 로직을 공유하거나, 특정 조건에 따라 컴포넌트를 렌더링하는 경우에 사용된다.

```js
const withLogging = (WrappedComponent) => {
  // withLogging은 컴포넌트를 감싸는 새로운 컴포넌트를 반환해요
  return class WithLogging extends React.Component {
    componentDidMount() {
      console.log(`Component ${WrappedComponent.name} 마운트!.`);
    }

    render() {
      return <WrappedComponent {...this.props} />;
    }
  };
};

const MyComponent = () => <div>My Component</div>;

const MyComponentWithLogging = withLogging(MyComponent);
```
