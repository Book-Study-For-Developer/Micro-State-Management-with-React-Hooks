## 05. 리액트 컨텍스트와 구독을 이용한 컴포넌트 상태 공유 

리액트 컨텍스트를 사용하면 각 하위 트리마다 서로 다른 값을 제공할 수 있고 구독은 불필요한 리렌더링을 막을 수 있다. 

리액트 컨텍스트와 구독을 합친 새로운 방법의 장점은 아래와 같다. 

- 컨텍스트는 하위 트리에 전역 상태를 제공할 수 있고 컨텍스트 공급자를 중첩으로 사용하는것이 가능하다. 컨텍스트를 사용하면 리액트 컴포넌트 생명 주기 내에서 useState와 같은 훅으로 전역 상태를 제어할 수 있다. 
- 반면 구독을 사용하면 단일 컨텍스트로는 불가능한 리렌더링 문제를 해결할 수 있다.

### 모듈 상태의 한계 
모듈 상태는 리액트 컴포넌트 외부에 존재하는 전역으로 정의된 싱글턴이기 때문에 **컴포넌트 트리나 하위 트리마다 다른 상태를 가질 수 없다는 한계**가 있다.

```jsx
const createStore = <T extends unknown>(initialState: T): Store<T> => {
  let state = initialState;
  const callbacks = new Set<() => void>();
  const getState = () => state;
  const setState = (nextState: T | ((prev: T) => T)) => {
    state =
      typeof nextState === "function"
        ? (nextState as (prev: T) => T)(state)
        : nextState;
    callbacks.forEach((callback) => callback());
  };
  const subscribe = (callback: () => void) => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };
  return { getState, setState, subscribe };
};
```

```jsx
const store = createStore({count : 0});

const Counter = () => {
  const [state , setState] = useStore(store);

  ...
}

const Component = () => {
  return (
    <>
      <Counter />
      <Counter />
    </>
  )
}
```

Counter 컴포넌트는 재사용 가능하기 때문에 Component는 두 개의 Counter 인스턴스를 가질 수 있다. 이렇게 Component는 동일한 상태를 공유하는 한 쌍의 카운터를 보여줄 수 있다. 

만일 위에서 만든 count값이 아닌 새로운 한쌍의 카운터를 추가하고 싶다면 아래와 같은 구조가 될 것 같다. 

```jsx
const store2 = createStore({count : 0});

const Counter2 = () => {
  const [state , setState] = useStore(store2);

  ...
}

const Component2 = () => {
  return (
    <>
      <Counter2 />
      <Counter2 />
    </>
  )
}
```

위와 같은 구조는 모듈 상태의 한계를 보여준다. Counter는 재사용 가능해야 하지만 모듈상태는 리액트 외부에 존재하기 떄문에 불가능하다. 

### 컨텍스트와 구독 패턴 사용하기

불필요한 리렌더링을 막고 전체 컴포넌트 트리에 대해 하나의 값만 제공해야한다는 문제점을 해결하기 위해서 아래와 같은 컨텍스트와 구독 패턴을 함께 사용할 수 있다. 

```ts
import { ReactNode, createContext, useContext, useRef, useMemo } from "react";
import { useSubscription } from "use-subscription";

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
    callbacks.forEach((callback) => callback());
  };
  const subscribe = (callback: () => void) => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };
  return { getState, setState, subscribe };
};

type State = { count: number; text?: string };

// context값에 store의 값을 넣는다.
const StoreContext = createContext<Store<State>>(
  createStore<State>({ count: 0, text: "hello" })
);

// 하위 트리에 서로 다른 스토어를 제공하기 위해 StoreProvider 구현
const StoreProvider = ({
  initialState,
  children,
}: {
  initialState: State;
  children: ReactNode;
}) => {
  // store객체가 첫 번째 렌더링에서 한 번만 초기화되게 만드는 데 사용된다.
  const storeRef = useRef<Store<State>>();
  if (!storeRef.current) {
    storeRef.current = createStore(initialState);
  }
  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
};

// 스토어 객체를 사용하기 위한 useSelector hook 
// 해당 부분에서 useContext와 useSubscription을 함께 사용함으로써 컨텍스트와 구독의 이점을 모두 누린다.
const useSelector = <S extends unknown>(selector: (state: State) => S) => {
  const store = useContext(StoreContext);
  // 이 부분이 현재는 useSyncExternalStore로 대체가 가능하다.
  return useSubscription(
    useMemo(
      () => ({
        getCurrentValue: () => selector(store.getState()),
        subscribe: store.subscribe,
      }),
      [store, selector]
    )
  );
};

// context를 활용해서 상태를 갱신하는 방법을 제공한다.
const useSetState = () => {
  const store = useContext(StoreContext);
  return store.setState;
};

// 사용 예시!
// 항상 외부에서 정의해야한다. 안그러면 useCallback을 활용해서 감싸줘야한다.
// ❓ 아마 리렌더링되면 함수의 새로운 참조값이 생기고 useSelector hook안에 있는 memo가 동작하지 않아서 그런 것 같다.
const selectCount = (state: State) => state.count;

// 특정 store에 연결되어 있지 않고 다른 스토어도 사용이 가능하다.
const Component = () => {
  const count = useSelector(selectCount);
  const setState = useSetState();
  const inc = () => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  );
};

const App = () => (
  <>
    <h1>Using default store</h1>
    <Component />
    <Component />
    <StoreProvider initialState={{ count: 10 }}>
      <h1>Using store provider</h1>
      <Component />
      <Component />
      <StoreProvider initialState={{ count: 20 }}>
        <h1>Using inner store provider</h1>
        <Component />
        <Component />
      </StoreProvider>
    </StoreProvider>
  </>
);

export default App;
```

위와 같이 App 컴포넌트에는 세 곳에 Component 컴포넌트가 포함되어 있다. 

1. StoreProvider 외부
2. 첫 번째 StoreProvider 컴포넌트 내부
3. 두 번째로 중첩된 StoreProvider 컴포넌트 내부

동일한 Store 객체를 사용하는 Component 컴포넌트는 동일한 count 값을 보여준다. 다른 Store를 사용하는 컴포넌트는 다른 count값을 보여준다.

위와 같이 컨텍스트로 인해 하위 트리에서 상태를 분리할 수 있고, 구독으로 인해 리렌더링 문제를 피할 수 있다.

