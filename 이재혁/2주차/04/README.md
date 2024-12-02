# 04. 구독을 이용한 모듈 상태 공유

- 전역 상태를 싱글톤과 유사하게 만들고 싶다? 모듈 상태를 써라! 그러면 메모리 관리에 좋다~
- 리액트에서 모듈 상태를 쓰기 위해서는 **구독**을 사용해야 한다.
- 모듈도 독립된 무언가

## 모듈 상태 살펴보기

- 모듈 상태 : 모듈 수준에서 정의된 변수
- 리액트와 유사한 형태로 모듈 상태를 쓰는 법을 살펴보자

```tsx
let state = {
  count: 0,
};

// get, set 함수 만들기
export const getState = () => state;
export const setState = (newVal) => {
  state = newVal;
};

// 리액트 처럼 업데이트 함수를 쓴다면?
export const setState = (newVal) => {
  state = typeof newVal === "function" ? newVal(state) : newVal;
};

// ======== 실제 사용 시 ========
setState((prev) => ({
  ...prev,
  count: prev.count + 999,
}));

// ======== container 방식으로 위의 정의 되었던 것들을 묶어서 필요한 set,get 함수를 return 한다

export const createContainer = (initVal) => {
  let state = initVal;
  // 생략.........
  return { getState, setState };
};
```

## 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

- 상태를 사용하고 변경하기 위해서는 useState 또는 useReducer를 써야함을 기억하자
- 두 개이상의 컴포넌트에서 쓰기 위해 외부에 상태를 두고 state에 담아 변경하는 방식은 옳지 않다. 완전 독립적인 하나의 컴포넌트에서 초기값을 할당하기 위해 쓰면 모를까...
- set과 같이 별도의 자료 구조를 추가해서 관리 가능하다.

```tsx
let count = 0;
const setStateFunctions = new Set<(count: number) => void>();

const Component1 = () => {
  const [state, setState] = useState(count);

  useEffect(() => {
    setStateFunctions.add(setState);
    return () => {
      setStateFunctions.delete(setState);
    };
  }, []);

  const inc = () => {
    count += 1;
    setStateFunctions.forEach((fn) => {
      fn(count);
    });
  };

  return (
    <div>
      {state} <button onClick={inc}>+1</button>
    </div>
  );
};
```

- `inc`에서 `forEach` 문을 돌며 set에 담긴 모든 seState를 돌며 count를 저장
- 공유하는 것 처럼보이게 하기? 컴포넌트가 많아지면 문제가 된다. set내의 값들이 전부 동일

## 기초적인 구독 추가하기

- 리액트에서 구독으로 키워드를 검색하면 [useSyncExternalStore](https://ko.react.dev/reference/react/useSyncExternalStore) 훅이 나온다.
- 참고하면 좋을 듯!
- 구독은 기본적으로 상태가 변경되었을 때 구독된 모든 곳에 "나 변경되었소!" 하고 알리는 것? 이라 생각한다.

```tsx
type Store<T> = {
  getState: () => T;
  setState: (action: T | ((prev: T) => T)) => void;
  subscribe: (callback: () => void) => () => void;
};
const createStore = <T extends unknown>(initialState: T): Store<T> => {
  let state = initialState;
  const callbacks = new Set<() => void>();
  const getState = () => state;
  const setState = (nextState: T | ((prev: T) => T)) => {
    state =
      typeof nextState === "function"
        ? (nextState as (prev: T) => T)(state)
        : nextState;
    callbacks.forEach((callback) => callback()); // 2. 상태가 변경되면 구독된 콜백 호출
  };
  const subscribe = (callback: () => void) => {
    callbacks.add(callback); // !. 구독하면 알림용 콜백을 저장
    return () => {
      callbacks.delete(callback);
    };
  };
  return { getState, setState, subscribe };
};
```

```js
// ==================== createStore 사용하기 ====================

import { createStore } from "./어딘가";

// 1. 기본 사용
const store = createStore({ count: 0 });
console.log(state.getState()) // 초기값인 {count: 0} 이 반환
store.setState({ count: 1 }); // {count:1}로 변경
store.subscribe(...(원하는 콜백))

```

```tsx
// 2.  리액트 스럽게 사용
const useStore = (store) => {
  const [state, useState] = useState(store.getState());

  useEffect(() => {
    setState(store.getState());

    //????
    return store.subscribe(() => {
      setState(store.getState());
    });
  }, [store]);
};
```

- ⚠️ 클린업 함수에서 `setState()` 를 한번 호출하는 이유가 `useEffect`가 뒤늦게 실행되어서 `store`가 이미 새로운 상태를 가지고 있을 가능성이 있기 때문이다 라고하는데 왜 그런지 텍스트 상으로 하는 이해가 잘 안되는 군요.

- 이후에는 일반적으로 사용하는 `useState` 처럼 `useStore`를 사용하면 된다

## 선택자와 useSubscription 사용하기

- 앞서 만든 `useStore`는 객체 형태에서 일부만 변경되더라도 사용중인 모든 곳에서 리렌더링이 일어나서 최적화가 필요하다
- 그런 상황에서 쓸 수 있는 것이 `selector` 함수이다

- 아래와 같은 store를 만든다

```js
const store = createStore({ count1: 0, count2: 0 });
```

- `useStore`와 달리 추가로 `선택자(selector)`를 받고, `selector`사용해 상태의 범위를 지정한다.

```tsx
const useStoreSelector = <T, S>(store: Store<T>, selector: (state: T) => S) => {
  const [state, setState] = useState(() => selector(store.getState()));

  useEffect(() => {
    setState(selector(store.getState()));
    return store.subscribe(() => {
      setState(selector(store.getState()));
      setState(selector(store.getState())); //❗ 두 번 들어간건 잘못 들어간거겠죠?
    });
  }, [store, selector]);

  return state;
};

// ==================== useStoreSelector 사용하기 ====================

const store = createStore({ count1: 0, count2: 0 });

const Component = () => {
  const state = useStoreSelector(
    store,
    useCallback((state) => state.count1, []) // 새로 생성되는 것을 막아서 구독-해지 반복을 막기
  );

  const inc = () => {
    store.setState((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };
};
return (
  <button
    onClick={() => {
      inc();
    }}
  >
    +1
  </button>
);
```

- 이렇게 구현한 것은 useEffect의 영향으로 값을 반환하는게 꼬일 수 있다.
- 리액트 팀에서는 [use-subscription](https://www.npmjs.com/package/use-subscription)를 공식적으로 제공하고 있다.
