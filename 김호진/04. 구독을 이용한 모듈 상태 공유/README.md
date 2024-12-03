## 04. 구독을 이용한 모듈 상태 공유

```jsx
import { useEffect, useState } from "react";

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

const store = createStore({ count: 0 });

const useStore = <T extends unknown>(store: Store<T>) => {
  const [state, setState] = useState(store.getState());
  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });
    setState(store.getState()); //[1]
    return unsubscribe;
  }, [store]);
  return [state, store.setState] as const;
};

const Component1 = () => {
  const [state, setState] = useStore(store);
  const inc = () => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      {state.count} <button onClick={inc}>+1</button>
    </div>
  );
};

const Component2 = () => {
  const [state, setState] = useStore(store);
  const inc2 = () => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 2,
    }));
  };
  return (
    <div>
      {state.count} <button onClick={inc2}>+2</button>
    </div>
  );
};

const App = () => (
  <>
    <Component1 />
    <Component2 />
  </>
);

export default App;

```

❓ useStore를 사용한 부분에서 `setState(store.getState())` 해당 부분이 edge 케이스를 처리한 부분이라고 했는데 잘 이해가 안간다. store를 구독하고 있는 다른 A,B 컴포넌트에서 상태를 변경했을때 useEffect가 더 늦게 실행되는 케이스에 최신상태를 보장하기 위해 작성했다고 이해했는데 맞을까?

위 코드가 사실 useSyncExternalStore를 활용해서 zustand를 구현한것과 거의 비슷한 코드이다. zustand쪽 코드를 보면 아래와 같이 구성되어 있다.  

```tsx
...각종 타입들

const createStoreImpl: CreateStoreImpl = (createState) => {
  type TState = ReturnType<typeof createState>
  type Listener = (state: TState, prevState: TState) => void
  let state: TState
  const listeners: Set<Listener> = new Set()

  const setState: StoreApi<TState>['setState'] = (partial, replace) => {
    // TODO: Remove type assertion once https://github.com/microsoft/TypeScript/issues/37663 is resolved
    // https://github.com/microsoft/TypeScript/issues/37663#issuecomment-759728342
    
    // partial이 함수면 실행하여 다음 상태를 얻고, 아니면 값을 그대로 사용
    const nextState =
      typeof partial === 'function'
        ? (partial as (state: TState) => TState)(state)
        : partial
    
    // 상태가 실제로 변경되었을 때만 처리
    if (!Object.is(nextState, state)) {
      const previousState = state
      
     // replace가 true이거나 nextState가 객체가 아닌 경우 => 완전히 교체
     // 그렇지 않으면 => 기존 상태와 병합
      state =
        (replace ?? (typeof nextState !== 'object' || nextState === null))
          ? (nextState as TState)
          : Object.assign({}, state, nextState)
          
      // 모든 리스너에게 상태 변경을 알림
      listeners.forEach((listener) => listener(state, previousState))
    }
  }
  
  // 현재 상태 반환
  const getState: StoreApi<TState>['getState'] = () => state
  
  // 초기 상태 반환
  const getInitialState: StoreApi<TState>['getInitialState'] = () =>
    initialState
  
  // 상태 변경 구독
  const subscribe: StoreApi<TState>['subscribe'] = (listener) => {
    listeners.add(listener)
    // Unsubscribe
    return () => listeners.delete(listener)
  }

  const api = { setState, getState, getInitialState, subscribe }
  const initialState = (state = createState(setState, getState, api))
  return api as any
}

export const createStore = ((createState) =>
  createState ? createStoreImpl(createState) : createStoreImpl) as CreateStore 
```

타입을 제외하고 주요 구현체만 보면 거의 동일한것을 알 수 있다.