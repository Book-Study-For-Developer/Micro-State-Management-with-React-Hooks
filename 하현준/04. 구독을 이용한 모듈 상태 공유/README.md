## 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

```tsx
let count = 0;
// 상태를 변경하는 함수를 등록하는 Set
const setStateFunctions = new Set<(count: number) => void>();

const Component1 = () => {
  const [state, setState] = useState(count);

  // useEffect를 통해 상태 등록 함수를 등록
  useEffect(() => {
    setStateFunctions.add(setState);
    return () => {
      // 클린업 함수에서 추가한 상태를 제거
      setStateFunctions.delete(setState);
    };
  }, []);
  const inc = () => {
    count += 1;
    // 등록된 함수들을 모두 실행
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

// Component2에도 똑같은 과정을 통해 반복 코드 작성 수행
```

컴포넌트간의 상태를 공유하기 위해 **EventManager 패턴**을 사용하는 것으로 보인다.
이벤트를 등록하고 그것을 관리해주는 주체가 외부에 있고, 각각의 컴포넌트에서 이를 실행시켜 리렌더링을 통해 상태를 공유하는 형태로 작성한다.

이렇게 관리하는 방식의 단점은 누구나 보이듯이 **컴포넌트가 추가될 때 마다 똑같은 작업을 반복**해서 작업해야 한다는 것이다.

### 구독(subscribe) 방식을 사용해서 만들어보자.

기초적인 구독을 만드는 패턴의 코드라서 좋은 내용이라 들고 왔다.

```tsx
// Store에 대한 인터페이스
// ❓: type 대신 interface로 하면 더 괜찮을 것 같다.
type Store<T> = {
  getState: () => T;
  setState: (action: T | ((prev: T) => T)) => void;
  subscribe: (callback: () => void) => () => void;
};

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

// react에서 사용할 수 있도록 훅의 형태로 새로 Wrapping 해주기
// 직접 매번 사용하기 보다 이렇게 훅을 통해 wrapping해주면 사용하기가 쉬워진다.
const useStore = <T extends unknown>(store: Store<T>) => {
  const [state, setState] = useState(store.getState());

  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });

    // 책에서는 엣지케이스를 다루기 위해 해당 코드를 작성해놨다고 했다.
    // ❓: 어떤 엣지케이스를 다루기 위해서 이런 코드를 써놓았을지 궁금..
    // useEffect가 뒤늦게 실행돼서 새로운 상태를 가지고 있을 수 있기 때문이라는데 아직 완벽히 이해하지 못했다.
    // 내가 지금 이해한 바로는 이 코드가 없을 경우 상태가 변경되었지만 useEffect가 뒤늦게 실행되어 반영되지 않았을 경우를 대비해 여기서도 setState를 통해 상태를 맞춰주는 것으로 이해했다.
    setState(store.getState());
    return unsubscribe;
  }, [store]);

  return [state, store.setState] as const;
};

// 사용법
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
```

### selector를 이용해 특정상태만 변경하는 방법으로 고도화하자

core 로직은 그대로 둔채로 새로운 훅을 만든다.

```tsx
const store = createStore({ count1: 0, count2: 0 });

// selector를 사용한 커스텀 훅이다.
const useStoreSelector = <T, S>(store: Store<T>, selector: (state: T) => S) => {
  // selector를 통해서 특정 상태만 가져오는 작업이 추가로 들어갔다.
  const [state, setState] = useState(() => selector(store.getState()));

  useEffect(() => {
    // 구독하는 방법 또한 같이 변경이 들어갔다. selector를 통해 구독을 해지하는 방법이 필요하다.
    const unsubscribe = store.subscribe(() => {
      setState(selector(store.getState()));
    });

    setState(selector(store.getState()));
    return unsubscribe;
  }, [store, selector]);

  return state;
};

// 사용법
// 컴포넌트 내부에서 사용했을 경우 useCallback을 사용해야 했지만 외부 함수로 분리해서 그러한 작업을 제거하였다.
const selectCount2 = (state: ReturnType<typeof store.getState>) => state.count2;

const Component2 = () => {
  const state = useStoreSelector(store, selectCount2);

  const inc = () => {
    store.setState((prev) => ({
      ...prev,
      count2: prev.count2 + 1,
    }));
  };

  return (
    <div>
      count2: {state} <button onClick={inc}>+1</button>
    </div>
  );
};
```

이렇게 되면 count1, count2가 변할때 각각 사용되는 컴포넌트에서만 리렌더링이 일어나게 된다.

store나 selector가 변경되면 타이밍 이슈로 인해 불편함을 겪을 수 있어 `use-subscription`을 사용하라고 한다.

### use-subscription 패키지 뜯어보기

https://www.npmjs.com/package/use-subscription

> You may now migrate to [`use-sync-external-store`](https://www.npmjs.com/package/use-sync-external-store) directly instead, which has the same API as [`React.useSyncExternalStore`](https://reactjs.org/docs/hooks-reference.html#usesyncexternalstore). The `use-subscription` package is now a thin wrapper over `use-sync-external-store` and will not be updated further.

그런데 패키지 첫줄에 react의 useSyncExternalStore를 사용하라고 되어 있고, 더이상 업데이트하지 않겠다라고 되어있는걸 보니 쓰면 안될듯 ? ㅋㅋㅋ

내부 코드를 보니 useSyncExternalStore로 바뀌어 있다.

```tsx
/**
 * Copyright (c) Meta Platforms, Inc. and affiliates.
 *
 * This source code is licensed under the MIT license found in the
 * LICENSE file in the root directory of this source tree.
 *
 * @flow
 */

import { useSyncExternalStore } from "use-sync-external-store/shim";

// Hook used for safely managing subscriptions in concurrent mode.
//
// In order to avoid removing and re-adding subscriptions each time this hook is called,
// the parameters passed to this hook should be memoized in some way–
// either by wrapping the entire params object with useMemo()
// or by wrapping the individual callbacks with useCallback().
export function useSubscription<Value>({
  // (Synchronously) returns the current value of our subscription.
  getCurrentValue,

  // This function is passed an event handler to attach to the subscription.
  // It should return an unsubscribe function that removes the handler.
  subscribe,
}: {
  getCurrentValue: () => Value;
  subscribe: (callback: Function) => () => void;
}): Value {
  return useSyncExternalStore(subscribe, getCurrentValue);
}
```

이전 코드 찾아보면,,

https://github.com/facebook/react/pull/24289

해당 PR에서 수정된 것으로 보인다.

https://github.com/gaearon/react/blob/d96f478f8a79da3125f6842c16efbc2ae8bcd3bf/packages/use-subscription/src/useSubscription.js

```tsx
import {useDebugValue, useEffect, useState} from 'react';

export function useSubscription<Value>({
  getCurrentValue,
  subscribe,
}: {|
  getCurrentValue: () => Value,
  subscribe: (callback: Function) => () => void,
|}): Value {

  const [state, setState] = useState(() => ({
    getCurrentValue,
    subscribe,
    value: getCurrentValue(),
  }));

  let valueToReturn = state.value;

  // 구독이 업데이트 되면 리액트에서도 업데이트 시켜주는 코드
  if (
    state.getCurrentValue !== getCurrentValue ||
    state.subscribe !== subscribe
  ) {
    valueToReturn = getCurrentValue();

    setState({
      getCurrentValue,
      subscribe,
      value: valueToReturn,
    });
  }

  useDebugValue(valueToReturn);

  useEffect(
    () => {
      let didUnsubscribe = false;

      const checkForUpdates = () => {
        if (didUnsubscribe) {
          return;
        }
        const value = getCurrentValue();

        setState(prevState => {
          // 오래된 구독의 업데이트를 방지하기 위한 코드
          if (
            prevState.getCurrentValue !== getCurrentValue ||
            prevState.subscribe !== subscribe
          ) {
            return prevState;
          }

          // 이전 상태와 같으면 리렌더링을 하지 않도록 베일-아웃 처리를 한다.
          if (prevState.value === value) {
            return prevState;
          }

          return {...prevState, value};
        });
      };

      const unsubscribe = subscribe(checkForUpdates);

      checkForUpdates();

      return () => {
        didUnsubscribe = true;
        unsubscribe();
      };
    },
    [getCurrentValue, subscribe],
  );

  return valueToReturn;
}
```

그렇다면 useSyncExternalStore의 구현체를 살펴보자

https://github.com/facebook/react/blob/main/packages/use-sync-external-store/src/useSyncExternalStoreShimClient.js

```tsx
export function useSyncExternalStore<T>(
  subscribe: (() => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T {
  // 스냅샷을 통해 값을 얻어옴
  const value = getSnapshot();

  // value와 스냅샷을 가져오는 상태를 강제로 업데이트 하도록 구성
  // {inst} 는 {value, getSnapshot}
  const [{inst}, forceUpdate] = useState({inst: {value, getSnapshot}});

  useLayoutEffect(() => {
    inst.value = value;
    inst.getSnapshot = getSnapshot;

    // 변했으면 강제로 리렌더링시킨다.
    if (checkIfSnapshotChanged(inst)) {
      forceUpdate({inst});
    }
  }, [subscribe, value, getSnapshot]);

  useEffect(() => {
    if (checkIfSnapshotChanged(inst)) {
      forceUpdate({inst});
    }

    const handleStoreChange = () => {
      if (checkIfSnapshotChanged(inst)) {
        forceUpdate({inst});
      }
    };

    return subscribe(handleStoreChange);
  }, [subscribe]);

  useDebugValue(value);

  return value;
}

// 스냅샷 된 것이 변했는지 체크
function checkIfSnapshotChanged<T>(inst: {
  value: T,
  getSnapshot: () => T,
}): boolean {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !is(prevValue, nextValue);
  } catch (error) {
    return true;
  }
}
```

코드는 굉장히 간단하다. 외부에 정의된 상태를 리액트 라이프사이클에 얹혀서 강제로 업데이트시키는 동작을 한다.

그런데 위의 코드는 shim code로 16, 17버전에서도 동작하도록 만들기 위한 코드로 실제 18에서 구현된 코드는 아래에 있다.

> shim code란?
> 일반적으로 문제를 해결하는 새 API를 추가하여, 이미 존재하는 코드의 동작을 바로잡는 데 사용되는 코드 모음

https://github.com/facebook/react/blob/7670501b0dc1a97983058b5217a205b62e2094a1/packages/react-reconciler/src/ReactFiberHooks.js#L1687

해당 블로그에서 해석을 자세히 해놓았다.

- https://jser.dev/2023-08-02-usesyncexternalstore/#2-how-usesyncexternalstore-works-internally-in-react

사용 예시

- 당근/stackflow 내부 라이브러리에도 사용중
  - https://github.com/daangn/stackflow/blob/d54e4af961b13459446b1c48680896d262187097/integrations/react/src/__internal__/core/CoreProvider.tsx#L4
  - https://github.com/daangn/stackflow/blob/d54e4af961b13459446b1c48680896d262187097/core/src/makeCoreStore.ts#L384
