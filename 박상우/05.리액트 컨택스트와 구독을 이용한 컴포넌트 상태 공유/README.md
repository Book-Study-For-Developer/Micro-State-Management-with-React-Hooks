## 모듈 상태의 한계

모듈 상태는 리액트 컴포넌트 외부에서 전역으로 정의된 싱글턴이기 때문에 컴포넌트 마다 다른 상태를 가질 수 없다.

→ 동일한 컨텍스트에서는 같은 값을 참조한다.

→ 독립적인 새로운 상태를 만들기 위해서는 `Count2`, `Count3`와 같이 새로운 상태를 직접 만들어줘야 한다.

→ 이런 상황에서 리액트 컨텍스트를 활용하기 좋고, 별도의 Provider로 구분하여 별개의 상태를 만들 수 있다.

( props로 활용이 가능하지만, 모듈 상태가 props drilling을 개선하기 위한 방식이기 때문에 적절하지 않은 방식이다. )

```tsx
const store = create({ count : 0 });

const Counter = () => {
	const [state, setState] = useStore(store);
	const inc = () => {
		setState((prev) => ({
			...prev,
			count: prev.count + 1,
		}));
	};

	return (
		<div>{state.count} <button onClick={inc}>+1</button></div>
	)
}

// Counter와 별개의 Counter를 만들기 위해서 새로운 Count2를 만들어야한다.
const store2 = create({ count : 0 });

const Counter2 = () => {
	const [state, setState] = useStore(store2);

	...
}
```

## 컨텍스트가 필요한 시점

컨텍스트는 기본적으로 컨텍스트를 생성하는 시점에서 기본값과, Provider를 설정하는 곳에서 기본값을 설정할 수 있다.

```tsx
const ThemeContext = createContext('light');

const Component = () => {
  const theme = useContext(ThemeContext);
  return <div>Theme: {theme}</div>;
};

const AppWithProvider = () => {
  return (
    <ThemeContext.Provider value='dark'>
      <Component /> // Theme: dark
    </ThemeContext.Provider>
  );
};

const AppWithoutProvider = () => {
  return (
    <>
      <Component /> // Theme: light
    </>
  );
};
```

공급자가 있는 것과, 없는 것은 무슨 차이가 있을까?

→ 차이가 없다.

→ 공급자를 설정하는 것은 기본값을 재정의(Override)하기 위해서이다.

<br />

## 컨텍스트와 구독 패턴 사용하기

컨텍스트를 활용할때 가장 큰 문제는 리랜더링이 발생한다는 것이다.

구독을 활용할 때 큰 문제는 컴포넌트 트리에서 하나의 값만 제공한다는 문제가 발생한다.

→ 컨텍스트와 구독을 함께 사용함으로서 두 문제를 해소할 수 있다.

```tsx
// Store 정의
type State = { count: number; text?: state }

const StoreContext = createContext<Store<State>>(
	createStore<State>({ count: 0, text: 'Hello' });
)

// StoreProvider 구현
const StoreProviderr = ({
	initialState,
	children,
} : {
	initialState: State;
	children: ReactNode;
}) => {
	const storeRef = useRef<Store<State>>();

	if (!storeRef.current) {
		storeRef.current = createStore(initialState);
	}

	return (
		<StoreContext.Provider value={storeRef.current}>
			{ children }
		</StoreContext.Provider>
	)
}
//useSelector
const useSelector = <S extends unknown>(
	selector: (state: State) => S
) => {
	const store = createStore(StoreContext);

	return useSubscribtion(
		useMemo(() => ({
			getCurrentValue: () => selector(store.getState());
			subscribe: store.subscribe,
		}),
		[store, selector]);
		)
	)
}

// useSetState
const useSetState = () => {
	const store = useContext(StoreContext);

	return store.setState;
}
```

```tsx
const selectCount = (state: State) => state.count;
// 외부에 선언하지 않으면 selectCount를 참조하는 useSelector 훅 내부에서 useMemo가 매번 갱신되기 때문에 계속해서 불필요한 재구독이 발생한다.
// 내부에서 사용하는 경우 useCallback을 추가한다.

const Component = () => {
	const count = useSelector(selectCount);
	const setState = useSetState();

	...
}
```

컴포넌트 특정 스토어와 연결되어 있는 것이 아니기 때문에 여러 스토어에 대해서 재사용가능하다.

```tsx
const App = () => (
  <>
    <h1>Using default store</h1>
    <Component /> // + 1
    <Component /> // + 1
    <StoreProvider initialState={{ count: 10 }}>
      <h1>Using store provider</h1>
      <Component /> // +11
      <Component /> // +11
      <StoreProvider initialState={{ count: 20 }}>
        <h1>Using inner store provider</h1>
        <Component /> // +22
        <Component /> // +22
      </StoreProvider>
    </StoreProvider>
  </>
);
```
