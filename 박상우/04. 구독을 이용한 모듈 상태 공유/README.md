# 4. 구독을 이용한 모듈 상태 공유

## 모듈 상태 정의하기

모듈 상태는 모듈 수준에서 정의된 변수

```tsx
export const createContainer = (initialState) => {
	let state - initialState;

	const getState = () => state;
	const setState = (nextState) => {
		state = typeof nextState === 'function' ? nextState(state) : nextState
	}

	return { getState, setState };
}
```

<br />

## 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

- 전역 상태가 필요하다면 Context를 사용하는 것 보다 모듈 상태를 활용하는 것이 더 좋을 수 있음
  → 하지만 리랜더링 최적화를 직접 처리해야한다.
- 일반 변수를 전역에서 선언하여 여러 컴포넌트가 각 컴포넌트 내부에서 useState의 초기값으로 활용하여 사용한다면, 각 컴포넌트에서는 문제가 없지만, 서로 다른 컴포넌트 간 값이 동기화되지 않는 문제가 있다.  
  → 컴포넌트 생명 주기를 고려하면서 컴포넌트 외부에 있는 모듈 수준에서 setState를 관리해야한다.

  ```tsx
  let count = 0;
  const setStateFunctions = new Set<(count: number) => void>();

  const Component1 = () => {
    const [state, setState] = useState(count);
    useEffect(() => {
      // 마운트 될 때, setState를 setStateFunctions에 추가
      setStateFunctions.add(setState);
      return () => {
        // 언마운트 될 때, 더 이상 해당 컴포넌트의 setState를 활요하지 못하도록 setStateFunctions에서 제거
        setStateFunctions.delete(setState);
      };
    }, []);
    const inc = () => {
      // 값 변경
      count += 1;
      // setStateFunctions 내부의 모든 setState를 실행함으로서 다른 컴포넌트의 상태를 갱신
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

  ### 기초적인 구독 추가하기

  ```tsx
  // 구독의 일반적인 사용법
  const unsubscribe = store.subscribe(() => {
    console.log("");
  });
  ```

  → store가 갱신될 때 마다, 콜백 함수가 호출된다.
  구독을 활용하여 모듈 상태 구현

  ```tsx
  type Store<T> = {
  	getState: () => T;
  	setState: (action: T | ((prev: T) => T)) => void;
  	subscribe: (callback: () => void) => () => void;
  }

  const createStore = <T extends unknown>(
  	initialState: T
  ) : Store<T> => {
  	let state = initialState;
  	const callbacks = new Set<() => void>();
  	const getState = () => state;
  	const setState = (nextState: T | ((prev: T) => T)) => {
  		state = typeof nextState === 'function'
  			? (nextState as (prev: T) => T)(state)
  			: nextState;

  		callbacks.forEach((callback) => callback());
  	};

  	const subscribe = (callback: () => void) {
  		callbacks.add(callback);
  		return () => {
  			callbacks.delete(callback);
  		}
  	};

  	return { getState, setState, subscirbe };
  }
  ```

<br />

## 선택자와 useSubscription 사용하기

모듈 상태를 만든 것에서 객체 상태에서 원하는 값만 선택할 수 있는 선택자를 받아 상태의 범위를 지정하도록 한다.

```tsx
const useStoreSelector = <T, S>(
	store: Store<T>,
	selector: (state: T) => S
) => {
	const [state, setState] = useState(() => selector(store.getState()));

	useEffect(() => {
		const unsubscribe = store.unsubscribe(() => {
			setState(selector(store.getState());
			setState(selector(store.getState());
		});
		setState(selector(store.getState());

		return unsubscribe;
	}, [store, selector])

	return state;
}

const Component1 = () => {
  const state = useStoreSelector(
    store,
    // useCallback을 사용하지 않으면, useStoreSelector의 useEffect 두번째 인자에 setState가 있어 랜더링 때마다 store.unsubscribe/subscribe를 하게 된다.
    useCallback((state) => state.count1, [])
  );
  const inc = () => {
    store.setState((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };
  return (
    <div>
      count1: {state} <button onClick={inc}>+1</button>
    </div>
  );
};
```

→ useStoreSeletor 훅에서 selector가 변경될 때, useEffect 보다 늦게 실행되기 때문에 재구독 전까지 이전 값을 활용하는 문제가 있을 수 있음  
→ 직접 구현하는 것 보다 React팀에서 제공하는 use-subscription이라는 공식적인 훅을 사용하여 활용하는 것이 낫다.
