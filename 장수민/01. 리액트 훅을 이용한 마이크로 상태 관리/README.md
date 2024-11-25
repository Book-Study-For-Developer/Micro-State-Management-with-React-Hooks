## 리액트 훅을 이용한 마이크로 상태 관리

> 리액트에서 상태는 사용자 인터페이스를 나타내는 모든 데이터를 말한다.

리액트 훅이 나오고 나서는 무조건 중앙 집중형 상태 관리 라이브러리를 사용하는 것보다 목적 지향적인 방법으로 여러 리액트 훅 기반 라이브러리를 가져오게 됐다.

하지만, 📌 **목적 지향적인 방법으로 처리할 수 없는 상태도 있기에 여전히 범용적인 상태 관리가 필요하다.** 📌

범용적인 상태 관리가 필요한 작업 비율은 어플리케이션마다 다른데 예를 들어,

- 서버 상태를 주로 다루는 어플리케이션이라면 하나 또는 소수의 전역 상태만 필요하다.
- 반면 풍부한 그래픽을 제공하는 어플리케이션은 많은 전역 상태가 필요하다.

> [!NOTE]
> 그래서, 범용적인 상태 관리를 위한 방법은 가벼워야 하고 개발자는 요구사항에 따라 적절한 방법을 선택할 수 있어야 한다. 이를 마이크로(가벼운) 상태 관리라고 한다.

---

### 리액트 훅 사용하기

마이크로 상태 관리를 하기 위해서는 리액트 훅이 필수다.

- `useState`: 지역 상태를 생성하는 기본적인 함수로, 로직을 캡슐화하고 재사용 가능하다는 특징이 있다.
- `useReducer`: useState 훅처럼 지역 상태를 생성할 수 있고, useState를 대체하는 용도로 자주 사용한다.
- `useEffect`: 리액트 렌더링 프로세스 바깥에서 로직을 실행할 수 있도록 도와준다. 특히 전역 상태를 다루기 위한 상태 관리 라이브러리를 개발할 떄 중요한데, 그 이유는 리액트 컴포넌트 생명 주기와 함께 작동하는 기능을 구현할 수 있기 때문이다.

---

### useState 사용하기

#### 베일아웃

```tsx
// 버튼을 처음 누르고 나서 이후로부터는, 동일한 상태값이 지속되므로 베일아웃(bailout)이 발생
// 즉, 리렌더링이 발생하지 않는 경우가 일어난다.
const Component = () => {
  const [count, setCount] = useState<Number>(0);

  return (
    <div>
      {count}
      <button onClick={() => setCount(1)}>Set Count to 1</button>
    </div>
  );
};
```

#### 빠르게 클릭한 만큼, 혹은 여러번 setCount를 호출한 만큼 숫자 안올라가는 이유 알기

```tsx
const Component = () => {
  const [count, setCount] = useState<Number>(0);

  return (
    <div>
      {count}
      {/* <button onClick={() => setCount(count + 1)}> */}
      <button onClick={(c) => setCount(c + 1)}>Set Count to {count + 1}</button>
    </div>
  );
};
```

> [!WARNING]
> 리액트에서 setState를 사용해서 상태를 업데이트할 경우에, 업데이트된 상태는 비동기적 특성 때문에 즉시 반영되지 않는다. 그리고, 효율적으로 렌더링하기 위해서 여러 번의 상태값 변경 요청을 배치(batch)로 처리한다. 결국, 리렌더링 요청에 따른 비용을 최소화하기 위해서 일괄적인 업데이트를 한다.

> [!NOTE]
> ❓ 그럼 항상 함수 업데이트를 해야 할까? ❓ - 나쁠 건 없지만 항상 그래야만 하는 것은 아니다. 리액트는 클릭과 같은 의도적인 사용자 액션에 대해 항상 다음 클릭 전에 상태가 업데이트 되도록 보장한다. 즉, 오래된 상태를 볼 위험은 없다~라고 공식 문서에 적혀 있다. 다만 동일한 이벤트 내에서 여러 업데이트를 수행하는 경우에는 함수 업데이트를 사용하는 것이 합리적이다.

---

### useReducer 사용하기

```tsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'SET_TEXT':
      if (!action.text) {
        // 베일아웃
        // 아래처럼 state 객체 자체를 반환해야 베일아웃이 일어나지 않는데,
        // 만약 { ...state }을 반환한다면 새로운 객체를 복사하게 되므로 결국 리렌더링이 촉발된다.
        return state;
      }
      return { ...state, text: action.text };
    default:
      throw new Error('unknown action type');
  }
};

const Component = () => {
  const [state, dispatch] = useReducer(reducer, { text: 'hi' });

  return (
    <div>
      <input value={state.text} onChange={(e) => dispatch({ type: 'SET_TEXT', text: e.target.value })}>
    </div>
  );
};
```

---

### useState와 useReducer의 유사점과 차이점

**유사점 - 상호 구현이 가능하다.**

```tsx
// useReducer로 useState 구현하기
const reducer = (prev, action) => (typeof action === 'function' ? action(prev) : action);

const useState = (initialState) => {
  const [state, dispatch] = useReducer(reducer, initialState);
  return [state, dispatch];
};
```

```tsx
// useState로 useReducer 구현하기
const useReducer = (reducer, initialState) => {
  const [state, setState] = useState(initialState);
  const dispatch = (action) => setState((prev) => reducer(prev, action));
  return [state, dispatch];
};
```

**차이점 - useReducer만 reducer와 init(지연 초기화를 위한 콜백 함수)을 훅이나 컴포넌트 외부에서 정의할 수 있다.**

```tsx
// useReducer
const init = (count) => ({ count });
const reducer = (prev, delta) => ({ ...prev, count: prev.count + delta });

const ComponentWithUseReducer = ({ initialCount }) => {
  const [state, dispatch] = useReducer(reducer, initialCOunt, init);

  // ...
};
```

```tsx
// useState
// 큰 차이점은 아니지만 useState는 외부에서 정의된 init과 reducer 함수를 호출해야 한다.
// 일부 인터프리터나 컴파일러는 인라인 함수 없이도 최적화가 더 잘 될 수 있다.
const ComponentWithUseState = ({ initialCount }) => {
  const [state, setState] = useState(() => init(initialCount));
  const dispatch = (delta) => setState((prev) => reducer(prev, delta));

  // ...
};
```

> ❓ 인라인 함수 없이도 최적화가 더 잘 될 수 있다라는 말이 무엇일까 ❓

JavaScript V8 엔진의 최적화 기법 중에는 Inlining(인라이닝)이라는 기법이 있다. 인라이닝은 함수 호출의 오버헤드를 줄이기 위해 사용되는 컴파일러 최적화 기법이다. 이 과정에서는 함수 호출 대신 함수의 본문으로 대체하고, 실행 시간을 단축시키는 데 도움이 된다고 한다.

```js
function add(a, b) {
  return a + b;
}

function main() {
  let result = add(2, 3);
  console.log(result);
}

// -------------------
// 위 코드 대신 인라이닝 기법을 활용하면,

function main() {
  let a = 2;
  let b = 3;
  let result = a + b;
  console.log(result);
}
```

이러한 인라이닝 기법을 통해 **1. 함수 호출과 반환에 소요되는 시간을 줄여, 전체 실행 시간을 감소시키고**, **2. 오히려 작은 함수의 경우 호출 자체가 함수 내부 로직보다 더 많은 시간을 소요할 수 있는데 이러한 오버헤드를 줄여준다.**

그렇기 때문에 인라인 함수 없이도 최적화가 더 잘 될 수 있는 인터프리터나 컴파일러가 있다👈👈라는 말은, 결국 함수 호출에 드는 비용을 줄이기 위해 인라인 함수에서 함수를 호출하기보다 함수를 참조하는 편이 더 나을 수 있고 이런 의미에서 useReducer가 useState보다 다소 좋다고 얘기하는 것이 아닐까라는 생각이 든다.
