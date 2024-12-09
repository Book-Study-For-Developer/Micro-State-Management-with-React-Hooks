## 05. 리액트 컨텍스트와 구독을 이용한 컴포넌트 상태 공유

이번에는 리액트 컨텍스트와 구독을 합친 새로운 방법을 알아본다. 그러한 방법은 **하위 트리마다 다른 값을 가질 수 있고 불필요한 렌더링을 피할 수 있다**는 두 가지 이점을 모두 확보할 수 있다.

### 모듈 상태의 한계

전역으로 정의된 싱글턴이기 때문에 컴포넌트 트리나 하위 트리마다 다른 상태를 가질 수 없다는 한계가 있다.

```tsx
const createStore = (initialState) => {
  let state = initialState;

  const callbacks = new Set();
  const getState = () => state;
  const setState = (nextState) => {
    state = typeof nextState === "function" ? nextState(state) : nextState;
    callbacks.forEach((callback) => callback);
  };
  const subscribe = (callback) => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };

  return { getState, setState, subscribe };
};

const store = createStore({ count: 0 });
```

위의 코드에서 `createStore`를 사용해서 store를 만들면 리액트 컴포넌트 외부에 정의된다.

```tsx
const Counter = () => {
  const [state, setState] = useStore(store);
  const inc = () => {
    setState((prev) => {
      ...prev,
      count: prev.count + 1,
    });
  };

  return (
    <div>
      {state.count} <button onClick={inc}>+1</button>
    </div>
  );
};

// 두 Counter 인스턴스는 하나의 store를 공유하고 있으므로 동일한 상태를 나타낸다.
const Component = () => (
  <>
    <Counter />
    <Counter />
  </>
);
```

`createStore` 함수를 통해 다수의 store를 만들고 그러한 store를 참조하는 Counter를 생성할 수 있지만, 한 store 내에서 상태를 공유하기 때문에 재사용이 불가능하다는 점이 있다. 이것이 모듈 상태의 한계이다.

> [!WARNING]
> props에 store를 넘기면 Counter를 재사용할 수 있지 않을까라는 의문이 들지만, 결국 props drilling이 발생하지 않도록 모듈 상태를 소개했기 때문에 적절한 해결 방안이라고 할 수는 없다.

### 컨텍스트 사용이 필요한 시점

```tsx
const ThemeContext = createContext("light"); // provider의 value에 전달될 기본값

const Component = () => {
  const theme = useContext(ThemeContext);
  return <div>Theme: {theme}</div>;
};
```

```tsx
<ThemeContext.Provider value="this value is not used">
  <ThemeContext.Provider value="this value is not used">
    <ThemeContext.Provider value="this is the value used">
      <Component />
    </ThemeContext.Provider>
  </ThemeContext.Provider>
</ThemeContext.Provider>
```

컴포넌트 트리에 공급자가 없는 경우에는 기본값을 사용하고, 아닌 경우에는 가장 안쪽에 위치한 공급자의 값을 사용하게 된다. 공급자의 값을 override한다고 생각하면 편하다.

ThemeContext 사례처럼 적절한 기본값이 있다면 공급자를 여러개 사용할 이유가 없지만, 전체 컴포넌트 트리의 하위 트리에 대해 다른 값을 제공할 필요가 있다면 공급자와 컨텍스트를 중첩해서 사용해야 한다.

### 컨텍스트와 구독 패턴 사용하기

컨텍스트와 구독 각각의 문제를 다시 짚자면,

1. 하나의 컨텍스트를 사용해 전역 상태 값을 전파하는 것은 불필요한 리렌더링이 발생한다는 문제가 있다.
2. 구독을 이용한 모듈 상태는 이런 문제가 없지만 전체 컴포넌트 트리에 대해 하나의 값만 제공한다는 문제가 있다.

이 각각의 문제를 해결할 수 있게 컨텍스트와 구독을 함께 사용하는 코드를 구현하자.

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

구현 자체는 객체 상태 값을 다루는 store를 생성하는 코드와 동일한데, 이번에는 Context 값에 createStore를 사용한다.

```tsx
type State = { count: number; text?: string };

const StoreContext = createContext<Store<State>>(
  createStore<State>({ count: 0, text: "hello" })
);

// 하위 트리에 서로 다른 스토어를 제공하기 위해 StoreProvider 별도 구현
// 📌 이 때, useRef 훅을 사용하는 이유는 첫 번째 렌더링에서 한 번만 초기화되게 하기 위해서다.
const StoreProvider = ({
  initialState,
  children,
}: {
  initialState: State;
  children: ReactNode;
}) => {
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
```

이제 스토어 객체를 사용하기 위해 useSelector 훅을 구현한다.

```tsx
const useSelector = <S extends unknown>(Selector: (state: State) => S) => {
  const store = useContext(StoreContext);

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

const useSetState = () => {
  const store = useContext(StoreContext);
  return store.setState;
};
```

이제 구현한 내용을 가지고 Component를 만든다.

```tsx
// 외부에서 정의하지 않으면 useCallback 함수로 감싸서 리렌더링때마다 초기화되는 것을 방지하는 추가 작업이 필요하다.
const selectCount = (state: State) => state.count;

const Component = () => {
  const count = useSelector(selectCount);
  const setState = useSetState();
  const inc = () => {
    setState((prev) => {
      ...prev,
      count: prev.count + 1,
    });
  };

  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  );
};
```

여기서 중요한 점은 Component는 특정 스토어 객체에 연결되어 있지 않다는 점이다. 이전에 `useStore(store)`처럼 쓰는 것과는 다르다. 결국 다음과 같이 Component를 재사용할 수 있다.

```tsx
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
```
