## 4. 구독을 이용한 모듈 상태 공유

모듈 상태를 직접 정의하는 대신 상태와 상태에 접근할 수 있는 함수가 내부에 있는 컨테이너를 만들 수 있다. 그리고 컨테이너를 생성하는 함수를 만들 수 있다.

```tsx
export const createContainer = (initialState) => {
  let state = initialState;
  const getState = () => state;
  const setState = (nextState) => {
    state = typeof nextState === 'function' ? nextState(state) : nextState;
  };
  return { getState, setState };
};

import { createContainer } from '...';

const { getState, setState } = createContainer({
  count: 0,
});
```

### 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

전체 트리에서 전역 상태가 필요하다면 모듈 상태가 더 적합할 수 있다. 하지만 리액트 컴포넌트에서 모듈 상태를 사용하려면 리렌더링 최적화를 직접 처리해야 한다.

```tsx
let count = 0;

// count는 증가하지만 0으로 계속해서 초기화되기 때문에 리렌더링 X
// const Component1 = () => {
//   const inc = () => {
//     count += 1;
//   }

//   // return ...;
// }

const Component1 = () => {
  const [state, setState] = useState(count);
  const inc = () => {
    count += 1;
    setState(count);
  };

  // return ...;
};

const Component2 = () => {
  const [state, setState] = useState(count);
  const inc2 = () => {
    count += 2;
    setState(count);
  };

  // return ...;
};
```

Component1의 버튼을 클릭하더라도 Component2가 리렌더링되지 않는다. 또한 두 컴포넌트가 동일한 상태 값을 보여줄 것이라고 예상했지만 실제로 그렇지 않다. 불일치가 발생한다!

이를 해결하기 위해 컴포넌트 생명 주기를 고려(useEffect)하면서 컴포넌트 외부에 있는 모듈 수준에서 setState를 관리하기 위해 Set과 같은 별도의 자료구조에 추가할 필요가 있다.

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
      {state}
      <button onClick={inc}>+1</button>
    </div>
  );
};
```

### 기초적인 구독 추가하기

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
    callbacks.forEach((callback) => callback()); // 상태가 변경되면 구독된 콜백 호출
  };
  const subscribe = (callback: () => void) => {
    callbacks.add(callback); // 구독 시 알림용 콜백을 호출
    return () => {
      callbacks.delete(callback); // 콜백 삭제
    };
  };
  return { getState, setState, subscribe };
};

// createStore 사용
const store = createStore({ count: 0 });
console.log(state.getState());
store.setState({ count: 1 });
store.subscribe(...);

// 리액트에서 사용
const useStore = (store) => {
  const [state, useState] = useState(store.getState());

  useEffect(() => {
    setState(store.getState()); // 에지 케이스를 다루기 위한 코드? - useEffect가 늦게 실행돼서 이미 새로운 store값을 가지고 있을지도 모르는 우려때문에 그렇다.

    return store.subscribe(() => {
      setState(store.getState());
    });
  }, [store]);
};
```

이러한 코드의 사용으로 Count1과 Count2의 어떤 버튼을 눌러도 동기화가 잘 된다.

### 선택자

위에서 만든 useStore는 상태 객체 전체를 반환하므로, 일부분만 변경되더라도 모든 useStore 훅에 전달되기 때문에 불필요한 리렌더링을 발생시킬 수 있다.

그래서, 컴포넌트가 필요로 하는 상태의 일부분만 반환하는 선택자(selector)를 도입할 수 있다.

```tsx
const store = createStore({ count1: 0, count2: 0 });

const useStoreSelector = <T, S>(store: Store<T>, selector: (state: T) => S) => {
  const [state, setState] = useState(() => selector(store.getState())); // 원하는 상태만 반환하도록 하는 지연 콜백 삽입

  useEffect(() => {
    setState(selector(store.getState())); // 엣지 케이스 다루기

    return store.subscribe(() => {
      setState(selector(store.getState()));
    });
  }, [store, selector]);

  return state;
};

// App 컴포넌트 아래 Component1과 Component2가 있다고 가정하면 count1과 관련이 있는 Component1만 선택적으로 리렌더링된다.
const Component1 = () => {
  const state = useStoreSelector(
    store,
    useCallback((state) => state.count1, [])
  );

  const inc = () => {
    store.setState((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };

  // return ...
};
```

### useSubscription

store 또는 selector가 변경될 때 주의할 점이 있는데, useEffect는 조금 늦게 실행되기 때문에 재구독될 때까지는 갱신되기 이전 상태 값을 반환한다.

그래서 리액트 팀은 use-subscription이라는 공식적인 훅을 제공한다.

### useSyncExternalStore

🧐 왜 우리는 이 훅을 볼 일이 그렇게 많지 않을까? - 리액트 18에서 제공하는 concurrent feature 기능을 어플리케이션에 사용하지 않기 때문이다. 또한, 각 프로젝트에서 사용하는 mobx, redux, zustand과 같은 external store에서 useSyncExternalStore 사용이 필요없도록 설정해 놓았기 떄문이기도 하다!

> [!NOTE]
> concurrent feature란, 렌더링 타이밍 도중 사용자의 입력과 같은 즉각적으로 UI에 적용되어야 하는 부분에 대해 우선순위를 정해 렌더링할 수 있는 기능이다.

> [!NOTE]
> external store라고 표현하게 된 이유는, 이러한 상태 관리 흐름은 리액트에서 관찰하지 않기 때문이다. 반대로, internal store에는 useState, useReducer, context, props가 있을 수 있다.

🧐 그럼 이 훅은 왜 나왔을까? - concurrent feature에서 발생하는 Tearing이라는 이슈때문이다. 리액트에서 말하는 tearing은 의도치 않게 상태 불일치로 서로 일치하지 않는 시점의 UI가 렌더링되는 것을 의미한다.

> [!CAUTION]
> useEffect는 컴포넌트가 mount, update, unmount 될 때 콜백이 실행된다고 알려져 있다. 그런데 이 말이 조금 모호한데, 실제로는 컴포넌트의 렌더링과 동시에 일어나는 게 아니라 그 후에 일어난다.

```tsx
const App = () => {
  const [input, setInput] = useState('');

  console.log('1');

  useEffect(() => {
    console.log('mount');
    return () => {
      console.log('unmount');
    };
  }, [input]);

  console.log('2');

  return (
    <div>
      <input value={input} onChange={(e) => setInput(e.target.value)}></input>
    </div>
  );
};
```

콘솔에 나타나는 순서는 1, 2, unmount, mount이다. useEffect로 관리해야 할 사용자 입력과 관련된 UI가 리렌더링하면서 수행하는 로직이 external store를 업데이트 하는 일이라면 Tearing이라는 상황이 일어나기가 더 쉽다.

이름에서부터 externalStore와 동기화하겠다는 의지가 잘 드러나는 useSyncExternalStore의 사용 방법은 다음과 같다.

```tsx
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);

// subscribe는 store가 변경되었을 때 호출할 콜백 함수
// getSnapshot: store의 현재 값을 리턴하는 함수
// getServerSnapshot: SSR 시 가지고 있던 스냅샷을 리턴하는 함수

// 실제로 사용해보기
const store = createStore({ count1: 0, count2: 0 });

const useStore = (store, selector) => {
  return useSyncExternalStore(
    store.subscribe,
    useCallback(() => selector(store.getState(), [store, selector])),
  );
};

const Component1 = () => {
  const state = useStore(
    store,
    (state) => state.count1
  );

  const inc = () => {
    store.setState((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
    }));
  };

  // return ...
};
```

엄청 간단하게 사용할 수 있으면서도, Tearing 문제가 발생하지 않도록 할 수 있다.
