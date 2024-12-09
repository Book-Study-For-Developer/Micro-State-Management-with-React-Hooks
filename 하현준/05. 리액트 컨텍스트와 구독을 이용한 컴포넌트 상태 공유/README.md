## 모듈 상태의 한계

모듈 상태는 외부에 존재하기 때문에 컴포넌트 트리에 속하지 않아서 트리마다 다른 상태를 가질 수 없는 한계가 있음.

이전 챕터에서 작성한 store를 만드는 함수

```tsx
// store를 만들어주는 core 로직
const createStore = <T extends unknown>(initialState: T): Store<T> => {
  // 초기 상태를 저장
  let state = initialState;

  // 콜백 함수들
  const callbacks = new Set<() => void>();

  // 상태를 가져오는 함수
  const getState = () => state;

  // 상태를 set하는 함수
  const setState = (nextState: T | ((prev: T) => T)) => {
    state =
      typeof nextState === "function"
        ? (nextState as (prev: T) => T)(state)
        : nextState;

    // 이때 등록하고 있는 콜백을 모두 실행한다.
    callbacks.forEach((callback) => callback());
  };

  // 이벤트를 구독하는 함수
  const subscribe = (callback: () => void) => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };

  return { getState, setState, subscribe };
};

const store = createStore({ count: 0 });

// 컴포넌트에서 사용예시
cosnt Counter = () => {
	cosnt [state, setState] = useStore(store);
	const inc = () => {
		setState((prev) => ({
			...prev,
			count: prev.count + 1,
		}));
	}

	return (
		<div>
			{state.count} <button onClick={inc}>+1</button>
		</div>
	);
}

// 이때 store2도 새로 만들어서 사용하고 싶다면?
// const store2 = createStore({ count: 0 });
// const [state, setState] = useStore(store2);
// 이런 식으로 전역 상태를 새로 정의해서 사용해야 할 것이다.

const Component = () => {
	<>
		<Counter />
		<Counter />
	<>
}
```

> 위 방식의 문제점 ?
> 스토어가 필요할 때마다 계속해서 만들어야 한다는 문제가 있다.
>
> 책에서 단기 해결책으로는 props로 넣는 방식을 예시로 알려주었다.
> 코드로 예를 들자면 다음과 같을 것 같다.
>
> 이렇게 props로 넘겨주는 방식으로 하면 store가 바뀌어도 재사용이 가능해진 것이다.
> 즉, `Counter - store`, `Counter2 - store2` 끼리 **결합되어 있던 것을 props를 통해 의존성을 제거**한 것.
>
> ```tsx
> // props를 추가..?
> cosnt Counter = ({ store }) => {
> 	cosnt [state, setState] = useStore(store);
>
> 	// ...
> }
> ```

### 컨텍스트의 사용 및 구독 패턴 활용하기

위에서 예시로 들었듯이 props를 사용하게 되면 props drilling이 발생하기 때문에 컨텍스트를 사용해야 한다.

일단 먼저 컨텍스트를 생성하고,

```tsx
type State = { count: number; text?: string };

const StoreContext = createContext<Store<State>>(createStore<State>({ count: 0, text: "hello" });
```

여러 상태를 공유할 수 있도록 제공자를 만든다.

```tsx
const StoreProvider = ({
  initialState,
  children,
}: {
  initialState: State;
  children: ReactNode;
}) => {
  // useRef를 통해 딱 한 번만 초기화되게 만든다.
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

책에서 작성되어 있는 코드

```tsx
// 책에 있는 코드
const useSelector = <S extends unknown>(selector: (state: State) => S) => {
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

const selectCount = (state: State) => state.count;

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
```

❓: 책에서 구현한 Component는 특정 객체에 연결돼 있지 않다는데 `useSelector`, `useSetState`내부에 결국 직접적으로 store를 사용 중인데 왜 연결돼 있지 않다고 표현했을까?

근데 위에서 작성한 코드를 보고 **store가 결국 StoreContext와 엮여있다는 생각이 들어서 조금 더 일반적인 함수**로 만들어보겠다.

### **내가 변형한 최종 코드 형태**

```tsx
// Store.ts
// 전역 상태 라이브러리 직접 구현한 파일

type Store<T> = {
  getState: () => T;
  setState: (action: T | ((prev: T) => T)) => void;
  subscribe: (callback: () => void) => () => void;
};

// store를 만들어주는 core 로직
const createStore = <T,>(initialState: T): Store<T> => {
  // 초기 상태를 저장
  let state = initialState;

  // 콜백 함수들
  const callbacks = new Set<() => void>();

  // 상태를 가져오는 함수
  const getState = () => state;

  // 상태를 set하는 함수
  const setState = (nextState: T | ((prev: T) => T)) => {
    state =
      typeof nextState === "function"
        ? (nextState as (prev: T) => T)(state)
        : nextState;

    // 이때 등록하고 있는 콜백을 모두 실행한다.
    callbacks.forEach((callback) => callback());
  };

  // 이벤트를 구독하는 함수
  const subscribe = (callback: () => void) => {
    callbacks.add(callback);
    return () => {
      callbacks.delete(callback);
    };
  };

  return { getState, setState, subscribe };
};

// store set하는 함수
const useSetState = <T,>(store: Store<T>) => {
  return store.setState;
};
```

```tsx
// StoreContext.ts
// 컨텍스트 + Store를 같이 사용하게 해주는 파일

// store + context를 만들어주는 함수
const createStoreContext = <S,>(initialState: S) => {
  return createContext(createStore<S>(initialState));
};

// useSyncExternalStore로 변경해서 사용
// 외부의 store를 특정 select로 리액트 라이프사이클과 연동하는 함수
const useSelector = <T, S>(store: Store<T>, selector: (state: T) => S): S => {
  return useSyncExternalStore(store.subscribe, () =>
    selector(store.getState())
  );
};

const StoreProvider = ({
  StoreContext,
  initialState,
  children,
}: {
  StoreContext: Context<Store<State>>;
  initialState: State;
  children: ReactNode;
}) => {
  // useRef를 통해 딱 한 번만 초기화되게 만든다.
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

```tsx
// useCounter.ts
// Counter 컴포넌트에 사용될 커스텀 훅
const selectCount = (state: State) => state.count;

const useCounter = () => {
  // Counter 컴포넌트에 쓰일 컨텍스트 꺼내오기
  const store = useContext(CounterStoreContext);
  // count 상태만 가져와 성능 최적화하기
  const count = useSelector(store, selectCount);
  const setState = useSetState(store);

  // 숫자 증가시키는 함수 구현하기
  const inc = useCallback(() => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 1,
    }));
  }, [setState]);

  return {
    count,
    inc,
  };
};
```

```tsx
// CounterContextProvider.tsx
const CounterStoreContext = createStoreContext<State>({
  count: 0,
  text: "hello",
});

// Counter에 쓰일 제공자(Provider)
export const CounterStoreProvider = ({
  children,
  initialState = { count: 10 },
}: {
  children: ReactNode;
  initialState?: State;
}) => {
  return (
    <StoreProvider
      StoreContext={CounterStoreContext}
      initialState={initialState}
    >
      {children}
    </StoreProvider>
  );
};
```

```tsx
// Counter.tsx
const Counter = () => {
  // 커스텀 훅에서 사용되는 것들만 꺼내와 사용하기
  const { count, inc } = useCounter();

  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  );
};

// App.tsx
// 여러 곳에 쓰이는 컴포넌트 형태
const App = () => (
  <>
    <h1>Using default store</h1>
    <Counter />
    <Counter />
    {/* 초기 상태 10으로 설정 */}
    <CounterStoreProvider initialState={{ count: 10 }}>
      <h1>Using store provider</h1>
      <Counter />
      <Counter />
      {/* 초기 상태 20으로 설정 */}
      <CounterStoreProvider initialState={{ count: 20 }}>
        <h1>Using inner store provider</h1>
        <Counter />
        <Counter />
      </CounterStoreProvider>
    </CounterStoreProvider>
  </>
);

export default App;
```

책에서 소개한 것들을 코드에 직접 구현해보면서 실행해보니 Context + subscribe하는 형태를 자세히 알 수 있었다.

**위에 코드 살펴보고 틀린 것 있으면 말해주세요!! ☺️**
