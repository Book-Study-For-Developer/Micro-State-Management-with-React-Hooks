# 10. 사용 사례 시나리오 4: React-tracked

- 저자는 이 때를 기다리지 않았나싶다
- 앞에서 여러 전역 상태 라이브러리의 원리를 설명하고 후반에 저자의 라이브러리를 더 자세하게 소개하기 위한 빌드업이었음을…
- 유틸성 미들웨어의 사례까지 들고오면서 말이다 ㅋㅋ
- 저수준 레벨까지 지원하며 완벽한 호환에 대한 기대를!
- 그는 상태 관리의 신이었다..

## React Tracked

- 속성 감지 기반 → 자동 렌더링 최적화 (Valtio에서 찾아볼 수 있다)
- 다른 상태 관리 라이브러리와 함께 사용할 수 있다.
  - 주로 useState, useReducer를 사용
- 상태 관리 기능을 제공하지 않음 but 렌더링 최적화 기능을 제공 → **상태 사용 추적**
  - 그렇기에 상태 관리를 위한 라이브러리와 같이 사용
- Context API를 사용할 경우 별도 Select 함수를 사용하지 않는 한 Provider 하위에서 Consumer의 역할을 하고 있는 컴포넌트들은 모두 리렌더링의 대상이 된다.
- React Tracked 에서는 useContext(SampleContext)를 대신해서 쓸 수 있는 useTrakced 훅을 제공한다.
- useTracked 는 상태를 프록시로 감싸고 사용을 추적한다.

```tsx
// use Context API
const useFirstName = () => {
  const [{ firstName }] = useContext(NameContext);
  return firstName;
};

// use React Tracked
const useFirstName = () => {
  const [{ firstName }] = useTracked();
  return firstName;
};
```

- Context API와 비교하였을 때 별차이는 없어보이지만 자동으로 렌더링을 최적화 한다.

### **Notes with React 18**

- This library internally uses `use-context-selector`, a userland solution for `useContextSelector` hook. React 18 changes useReducer behavior which `use-context-selector` depends on. This may cause an unexpected behavior for developers. If you see more `console.log` logs than expected, you may want to try putting `console.log` in useEffect. If that shows logs as expected, it's an expected behavior. For more information:
- ❓ 대충 이런 내용이 있는데 라이브러리와 useReducer 모두 `use-context-selector` 를 쓰고 있어서 충돌이 발생할 수 있다는 것 같다. 아마?

## 예시 사용법

- 그럼 어떻게 동작하는 걸까?

### 1. useState

1. 별도의 훅을 정의

```tsx
import { useState } from "react";

const useValue = () =>
  useState({
    count: 0,
    text: "hello",
  });
```

1. createContainer를 통해 Provider와 useTracked를 반환

```tsx
import { createContainer } from "react-tracked";

const { Provider, useTracked } = createContainer(useValue);
```

1. 컴포넌트에서 사용

```tsx
const Counter = () => {
  const [state, setState] = useTracked();

  const increment = () => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 1,
    }));
  };

  return (
    <div>
      <span>Count: {state.count}</span>
      <button type="button" onClick={increment}>
        +1
      </button>
    </div>
  );
};
```

### 2. React Redux

- 리액트 컨텍스트를 사용하지 않는 경우를 위해 저수준의 함수를 제공
  - 코어 로직이라 해야할까 아주 기본적인 기능을 제공?
  - 더 많은 책임을 지게함, 자유도가 높음?

```tsx
import { createTrackedSelector } from "react-tracked";

const useTrackedSelector = createTrackedSelector(useSelector);
```

- useSelector - 선택자 함수
- useTrackedSelector - 상태 사용 추적을 위해 전체 상태를 프록시로 감싸서 반환하는 훅
- 결과가 변경될 경우 리렌더링

```tsx
import { createStore } from 'redux'
import { Provider, useDispatch, useSelector } from 'react-redux'
import { createTrackedSelector } from 'react-tracked';

const initialState = { count: 0 };

const reducer = (state, action) => {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 };
    default:
      throw new Error(`Unhandled action type: ${action.type}`);
  }
};

// redux
const store = createStore(reducer)
// 이런 느낌???
  <Provider store={store}>
    <Counter />
  </Provider>

// react tracked
const useTrackedSelector = createTrackedSelector(useSelector)

const Counter = () => {
  // react tracked
  const { count } = useTrackedSelector();
  // redux
  const count = useSelector((state) => state.count)

  const dispatch = useDispatch();

  return (
    <div>
      <span>Count: {count}</span>
      <button type="button" onClick={() => dispatch({ type: 'increment' })}>
        +1
      </button>
    </div>
  );
};
```
