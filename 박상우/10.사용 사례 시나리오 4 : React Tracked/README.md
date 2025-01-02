## React Tracked 이해하기

React Tracked는 상태 관리를 제공하지는 않지만 렌더링 최적화 기능을 제공한다.

→ 상태 사용 추적

React Tracked에서 상태 사용 추적을 위해 useContext를 사용한다.

useContext는 일부 상태만 사용하더라도 다른 상태 변화에 의해 랜더링이 발생한다.

React Tracked는 상태를 추적하며 useContext의 대체제로 쓸 수 있는 useTracked라는 훅을 제공한다.

```jsx
const NameContext = createContext({
	{ firstName: 'React', lastName: 'Hooks' },
	() => {},
});

// useContext를 사용
const useFirstName = () => {
	const [{ firstName }] = useContext(NameContext);
	return firstName
}

// useTracked를 사용
const useTracked = () => {
	const [{ firstName }] = useTracked();
}
```

React-Tracked와 Valtio는 동일하게 상태 추적 기능을 활용하며, proxy-compare이라는 내부 라이브러리를 공통으로 활용하고 있다. ( [React-Tracked Code](https://github.com/dai-shi/react-tracked/blob/main/src/createTrackedSelector.ts), [Valtio Code](https://github.com/pmndrs/valtio/blob/main/src/react.ts#L112) )

→ 공통적으로 isChanged와 createProxy를 활용하고 있었다.

→ `createProxy` : 상태 객체를 Proxy로 감싸서 속성 접근을 추적, 인자로 WeakMap 인스턴스를 받는데, 해당 인스턴스에서 어떤 속성에 접근했는지 기록된다.

→ `isChanged` : 두 상태 객체를 비교. 이때 createProxy에 활용한 WeakMap도 함께 인자로 전달 받아, 변경 이력이 있는 속성에 대해서만 비교

<br />

## useState, useReduce와 함께 React Tracked 사용하기

### useState와 React Track 사용하기

```tsx
const useValue = () => useState({ count: 0. text: 'Hello' });

const StateContext = createContext<ReturType<typeof useValue> | null>(null);

const Provider = ({ children }: { children: ReactNode }) => {
	<StateContext.Provider value={useValue()}>
		{ children }
	</StateContext.Provider>
}

const useStateContext = () => {
	const contextValue = useContext(StateContext);
	if(contextValue === null) {
		throw new Error("Please use Provider");
	}

	return contextValue;
}

const Counter = () => {
	const [state, setState[ = useStateContext();
	const inc = () => {
		setState((prev) => ({
			...prev,
			count: prev.count + 1,
		});
	};

	return (
		<div>
			count: { state.count }
			<button type='button' onClick={inc}>+1</button>
		</div>
	);
};

count TextBox = () => {
	const [state, setState] = useStateContext();
	const setText = (text:string) => {
		setState((prev) => ({ ...prev, next }));
	}

	return (
		<div>
			<input
				value={state.text}
				onChange={(e) => setText(e.target.value)}
			/>
		</div>
	)
}

const App = () => (
	// 현재 상태 객체 내의 속성이 하나만 변경이 되어도 Context의 모든 속성이 업데이트 된다.
	// -> 랜더링 최적화를 위해서 상태를 더 작은 단위로 나눠야한다.
	<Provider>
		<div>
			<Counter/>
			<Counter/>
			<TextBox/>
			<TextBox/>
		</div>
	</Provider>
)
```

```tsx
// React Tracked를 사용하는 경우
import { createContainer } from 'react-tracked';

const useValue = () => useState({ count: 0. text: 'Hello' });

// Provider는 기존 useContext에서의 Provider와 동일한 역할을 한다.
// useTracked는 앞선 예제의 useStateContext()와 동일한 역할을 한다.
const { Provider, useTracked } = createContainer(useValue);

const Counter = () => {
	const [state, setState] = useTracked();

	...
}

const TextBox = () => {
	const [state, setState] = useTracked();

	...
}
```

<br />

### useReducer와 함께 React Tracked 사용하기

```tsx
const useValue = () => {
	type State = { count: number; text: string; }
	// 정의된 액션을 통해서 특정 상태를 변화시킨다.
	type Action =
		| { type: "INC" }
		| { type: "SET_TEXT"; text: string; }

	const [state, dispatch] = useReducer(
		(state: State, action: Action) => {
			if (action.type === 'INC') {
				return { ...state, count: state.count + 1 };
			}
			if (action.type === 'SET_TEXT') {
				return { ...state, text: action.text };
			}

			throw new Error('unknown action type');
		},
		{ count: 0, text: 'hello'}
	);

	// 상태 변화 확인
	useEffect(() => {
		console.log(state);
	}, [state]);

	return [state, dispatch] as const;
}

const { Providrer, useTracked } = createContainer(useValue);

const Counter = () => {
	const [state, dispatch] = useTracked();
	const inc = () => dispatch({ type: "INC"});

	...
}
```

React Tracked가 랜더링 최적화를 할 수 있는 이유는 use-context-selector라는 내부 라이브러리 때문이다. 라이브러리의 selector 함수를 사용해 컨텍스트 값을 구독하고, 이를 통해 기존 컨텍스트의 약점을 보완한다.

<br />

## React Redux와 함께 React Tracked 사용하기

> React Redux 내부에서 컨텍스트를 활용하지만 상태 전파하는데 활용하진 않는다. 상태 전파는 구독을 활용하여 전파된다.

React Tracked는 컨텍스트를 사용하지 않을 경우에 대해서 createTrackedSelector를 제공한다.

```tsx
const useTrackedState = createTrackedSelector(useSelector);
```

`useSelector` → 선택자 함수를 받아 함수의 결과를 반환하는 훅이며, 새롭게 정의한 useTrackedState는 상태를 추적하기 위해 proxy로 감싼 결과를 반환하는 훅이 된다.  

React Redux는 useSelector 훅을 제공하는데 이를 React Tracked와 함께 사용할 수 있다.

```tsx
import { createStore } from 'redux';
import { Provider, useDispatch, useSelector } from 'react-redux';
import { createTrackedSelector } from 'react-tracked';

type State = { count: number; text: string };
type Action = { type: 'INC' } | { type: 'SET_TEXT'; text: string };

const initialState: State = { count: 0, text: 'hello' };

const reducer = (state = initialState, action: Action) => {
  if (action.type === 'INC') {
    return { ...state, count: state.count + 1 };
  }
  if (action.type === 'SET_TEXT') {
    return { ...state, text: action.text };
  }
  return state;
};

const store = createStore(reducer);
const useTrackedState = createTrackedSelector<State>(useSelector);

const Counter = () => {
  const dispatch = useDispatch();
  const { count } = useTrackedState();
  const inc = () => dispatch({ type: 'INC' });
  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  );
};

const TextBox = () => {
  const dispatch = useDispatch();
  const state = useTrackedState();
  const setText = (text: string) => {
    dispatch({ type: 'SET_TEXT', text });
  };
  return (
    <div>
      <input value={state.text} onChange={(e) => setText(e.target.value)} />
    </div>
  );
};

const App = () => (
  <Provider store={store}>
    <div>
      <Counter />
      <Counter />
      <TextBox />
      <TextBox />
    </div>
  </Provider>
);

export default App;
```

<br />

## 향후 전망

React Tracked를 사용하는 방법

- 컨텍스트를 활용할 때 → createContext ( createTrackedSelector + use-context-selector )
- React Redux ( useSelector )를 사용할 때 → createTrackedSelector ( proxy-compare )

컨텍스트를 사용하는 경우 use-context-selector는 useContextSelector 훅을 제공한다. 기존 컨텍스트 동작( 컨텍스트 값이 바뀌면 해당 컨텍스트를 사용하는 모든 요소가 리랜더링 된다.)를 개선하기 위한 훅이다.

🚀 useContextSelector 훅에 대한 논의

http://github.com/facebook/react/pull/20646
