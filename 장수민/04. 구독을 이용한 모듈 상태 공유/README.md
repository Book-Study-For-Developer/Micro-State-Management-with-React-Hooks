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

// createStore 사용
const store = createStore({ count: 0 });
console.log(state.getState());
store.setState({ count: 1 });
store.subscribe(...);

// 리액트에서 사용
const useStore = (store) => {
  const [state, useState] = useState(store.getState());

  useEffect(() => {
    setState(store.getState()); // 에지 케이스를 다루기 위한 코드?

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

### useSubscription

store 또는 selector가 변경될 때 주의할 점이 있는데, useEffect는 조금 늦게 실행되기 때문에 재구독될 때까지는 갱신되기 이전 상태 값을 반환한다.

그래서 리액트 팀은 use-subscription이라는 공식적인 훅을 제공한다.
