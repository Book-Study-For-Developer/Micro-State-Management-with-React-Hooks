### Lifting Content Up

상위 컴포넌트의 상태에 의존하지 않는 자식 컴포넌트를 조상 컴포넌트로 끌어 올려서 리렌더링을 방지하는 기법

일반적으로 `children` props을 사용하여 구현한다.

```tsx
"use client";

import { FC, PropsWithChildren, useEffect, useState } from "react";

const GrandParentComponent = () => {
  return (
    <ParentComponent>
      <DescendantComponent /> // 여기에서 넣어줌
    </ParentComponent>
  );
};

export default GrandParentComponent;

const ParentComponent: FC<PropsWithChildren> = ({ children }) => {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount((prev) => prev + 1);
  };

  return (
    <div>
      <p>Parent</p>
      <button onClick={increment}>Increment</button>
      <ChildCompnent1 count={count}>{children}</ChildCompnent1>
    </div>
  );
};

interface Props {
  count: number;
}

const ChildCompnent1: FC<PropsWithChildren<Props>> = ({ count, children }) => {
  useEffect(() => {
    console.log("render ChildCompnent1");
  });
  return (
    <div>
      Count: {count}
      <p>{children}</p>
    </div>
  );
};

const DescendantComponent = () => {
  // 부모의 state를 전달받지 않음
  useEffect(() => {
    console.log("render DescendantComponent");
  });
  return <span>Descendant</span>;
};
```

### 싱글턴과 공유상태

싱글턴과 공유상태를 구분해서 사용해야 함

싱글턴 → 자바스크립트 메모리에서 참조하는 상태가 한 개

공유상태 → 싱글턴이 아닌 전역 상태, 메모리에서 단일 값일 필요는 없음

### Context API

- 여러 Provider를 사용해 조각조각 격리된 상태를 제공할 수 있다.
- **싱글턴 패턴을 피하고 각 하위트리에 서로 다른 값을 공급하기 위해 설계된 API임**
- 한계
  - 상태가 갱신될 때 모든 컨텍스트 Consumer가 리렌더링 된다.
  - 컨텍스트에 객체를 사용할 경우 객체는 한 속성만 바뀌어도 당연히 리렌더링 된다. → 리렌더링을 피하기 위해서는 작은 컨텍스트 조각을 여러개 만들어서 사용하면 된다. → 하지만 프로바이더 중첩을 피할수 없게 된다.

### 실습

- reduceRight, createElement를 사용해서 Provider 중첩을 막는 부분 새로웠음
- null을 통해 초기값을 반드시 넣어주어야 한다는 것을 명시한 것이 좋았음

```tsx
import { createContext, PropsWithChildren, useContext, useState } from "react";

export const useNumberState = (init = 0) => useState(init);

const createStateContext = <Value, State>(
  useValue: (init?: Value) => State
) => {
  const StateContext = createContext<State | null>(null);

  const StateProvider = ({
    initialValue,
    children,
  }: PropsWithChildren<{ initialValue: Value }>) => {
    return (
      <StateContext.Provider value={useValue(initialValue)}>
        {children}
      </StateContext.Provider>
    );
  };

  const useContextState = () => {
    const value = useContext(StateContext);
    if (value === null) {
      throw new Error("useContextState must be used within a StateProvider");
    }
    return value;
  };

  return [StateProvider, useContextState] as const;
};

export default createStateContext;
```

```tsx
"use client";

import { createElement } from "react";
import createStateContext, {
  useNumberState,
} from "./context/createStateContext";

const [CountProvider1, useCount1] = createStateContext(useNumberState);
const [CountProvider2, useCount2] = createStateContext(useNumberState);
const [CountProvider3, useCount3] = createStateContext(useNumberState);
const [CountProvider4, useCount4] = createStateContext(useNumberState);

const Page = () => {
  const providers = [
    [CountProvider1, { initialValue: 0 }],
    [CountProvider2, { initialValue: 10 }],
    [CountProvider3, { initialValue: 20 }],
    [CountProvider4, { initialValue: 30 }],
  ] as const;

  return providers.reduceRight(
    (children, [Component, props]) => createElement(Component, props, children),
    <Parent />
  );
};

export default Page;

const Counter1 = () => {
  const [count, setCount1] = useCount1();
  return <button onClick={() => setCount1(count + 1)}>{count}</button>;
};

const Counter2 = () => {
  const [count, setCount2] = useCount2();
  return <button onClick={() => setCount2(count + 1)}>{count}</button>;
};

const Counter3 = () => {
  const [count, setCount3] = useCount3();
  return <button onClick={() => setCount3(count + 1)}>{count}</button>;
};

const Counter4 = () => {
  const [count, setCount4] = useCount4();
  return <button onClick={() => setCount4(count + 1)}>{count}</button>;
};

const Parent = () => {
  return (
    <ul>
      <li>
        <Counter1 />
      </li>
      <li>
        <Counter2 />
      </li>
      <li>
        <Counter3 />
      </li>
      <li>
        <Counter4 />
      </li>
    </ul>
  );
};
```

### 모듈 상태 (module state)

모듈 수준에서 정의된 변수. 모듈은 ES 모듈 또는 파일을 의미함.

리액트에서 전역 상태 관리를 위해 모듈 상태를 구성할 때, 상태 변화에 따른 리렌더링을 유도하려면 useEffect를 사용하여 부수효과로 상태와 컴포넌트를 연결하고 구독을 구현할 수 있다.

리액트의 상태와 동일하게 모듈 상태를 불변적으로 갱신시켜야 한다.

### 구독의 일반적인 사용법

```tsx
const unsubscribe = store.subscribe(()=> {...})
```

Store의 구현

```tsx
export type Store<T> = {
  getState: () => T;
  setState: (action: T | ((prev: T) => T)) => void;
  subscribe: (callback: () => void) => void;
};

export const createStore = <T extends unknown>(initialState: T): Store<T> => {
  let state = initialState; // 모듈 변수
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

  return {
    getState,
    setState,
    subscribe,
  };
};
```

```tsx
import { createStore } from "./utils";

const store = createStore({ count: 0 });

export default store;
```

store의 상태 값과 갱신 함수를 튜플로 반환하는 사용자 정의 훅, useEffect를 사용하여 모듈 상태 갱신에 따른 리렌더링을 유발하고 있다.

```tsx
import { useEffect, useState } from "react";

const useStore = (store: any) => {
  const [state, setState] = useState(store.getState());
  useEffect(() => {
    // store의 모듈 상태와 state를 서로 연결하여 리렌더링을 유발
    const unsubscribe = store.subscribe(() => {
      setState(store.getState()); // 여기에서 구독 store.setState에서 실행될 callback -> 리렌더링 유발
    });
    setState(store.getState()); // 최초에 한번 실행
    return unsubscribe; // 언마운트시에 구독 해제
  }, [store]);

  return [state, store.setState];
};

export default useStore;
```

### 위 코드의 흐름 정리

1. store.setState 실행
2. store에서 모듈 상태인 state를 갱신하고 callbacks를 순회하면서 callback 호출
3. callback은 useStore에서 useEffect 내부의 subscribe를 통해 사전에 전달되었음
4. subscribe의 callback인 setState(store.getState())가 실행되면서 리렌더링 유발

### store의 상태 중 일부만 구독하는 useStoreSelector hook 구현

책에서 선택자 함수를 넘길때 useCallback을 사용하지 않으면 매번 구독해지와 재구독을 하게 된다고 하는데 확인해보자.

```tsx
import { useEffect, useState } from "react";
import { Store } from "../store/utils";

const useStoreSelector = <T, S>(store: Store<T>, selector: (state: T) => S) => {
  const [state, setState] = useState(selector(store.getState()));
  useEffect(() => {
    console.log("effect, 구독해지하고 다시 구독하는 중"); // effect가 실행되고 있는지 확인
    const unsubscribe = store.subscribe(() => {
      setState(selector(store.getState()));
    });
    setState(selector(store.getState()));
    return unsubscribe;
  }, [store, selector]);

  return state;
};

export default useStoreSelector;
```

재구독을 방지하려면 useCallback으로 감싸거나 컴포넌트 외부에서 selector 함수를 정의해야 합니다.

```tsx
const selector = (state: ReturnType<typeof store.getState>) => state.count2;

const Component1 = () => {
  // const state = useStoreSelector(store, (state) => state.count2);
  const state = useStoreSelector(store, selector);

  const state = useStoreSelector(
    store,
    useCallback((state) => state.count2, [])
  );

  const inc = () =>
    store.setState((prev) => ({ ...prev, count2: prev.count2 + 1 }));
  return (
    <div>
      <ul>
        <li>count1 : {state}</li>
        <li>
          <button onClick={inc}>inc</button>
        </li>
      </ul>
    </div>
  );
};
```

단 이러한 형태의 useEffect를 사용한 구독 구현은 useEffect가 조금 늦게 실행 되기 때문에 재구독될때까지는 갱신되기 이전 상태 값을 반환할 수 있다고 한다.

## rxjs, BehaviorSubject를 사용한 구독 구현

Ref : https://junwoo45.gitbook.io/learn-rxjs-korean/learn-rxjs/subjects

rxjs, subjects에 대한 설명보다는 해당 라이브러리에서 제공하는 유틸을 통해 구독을 간단하게 구현하는 부분만 살펴보겠습니다.

BehaviorSubject은 아래와 같은 특징을 가져 구독의 구현에 여러 이점을 가지고 있습니다.

- 생성 시 반드시 초기값을 설정해야 하며 구독자가 없더라도 초기값을 유지합니다.
- 항상 최신값을 저장합니다.
- 새로운 구독자는 구독 시점에 최신값을 즉시 보장받을 수 있습니다.
- 다중 구독을 지원합니다.

주요 메서드

- next(value): 새로운 값을 방출(emit)하여 상태를 업데이트합니다.
- subscribe(callback): 상태 변경을 구독하고, 방출된 값을 처리합니다.
- getValue(): 현재 저장된 값을 동기적으로 반환합니다.

```js
// 클래스 구현부
export class BehaviorSubject<T> extends Subject<T> {
  constructor(private _value: T) {
    super();
  }

  get value(): T {
    return this.getValue();
  }

  /** @internal */
  protected _subscribe(subscriber: Subscriber<T>): Subscription {
    const subscription = super._subscribe(subscriber);
    !subscription.closed && subscriber.next(this._value);
    return subscription;
  }

  getValue(): T {
    const { hasError, thrownError, _value } = this;
    if (hasError) {
      throw thrownError;
    }
    this._throwIfClosed();
    return _value;
  }

  next(value: T): void {
    super.next((this._value = value));
  }
}
```

라이브러리가 구독 기능을 제공하기 때문에 createStore에서 구독을 직접 구현할 필요가 없습니다.

```tsx
import { useEffect, useState } from "react";
import { BehaviorSubject } from "rxjs";

export type Store<T> = {
  getState: () => T;
  setState: (action: T | ((prev: T) => T)) => void;
  store$: BehaviorSubject<T>;
};

export const createStore = <T extends unknown>(initialState: T): Store<T> => {
  const store$ = new BehaviorSubject(initialState);
  const setState = (updater: T | ((prev: T) => T)) => {
    const currentState = store$.getValue();
    const newState =
      typeof updater === "function"
        ? (updater as (prev: T) => T)(currentState)
        : updater;
    store$.next(newState);
  };
  const getState = () => store$.getValue();
  // 대신 createStore가 store 자체를 반환하도록 해야합니다.
  return { store$, setState, getState };
};

export const useStore = <T,>(store: Store<T>): [T, Store<T>["setState"]] => {
  const [state, setState] = useState(store.getState());
  useEffect(() => {
    const subscription = store.store$.subscribe((newState) => {
      setState(newState);
    });
    return () => subscription.unsubscribe();
  }, [store]);

  return [state, store.setState];
};

export interface CounterState {
  count1: number;
  count2: number;
}

export const counterStore = createStore<CounterState>({
  count1: 0,
  count2: 0,
});
```

사용 예시

```tsx
"use client";

import useStore from "./hooks/useStore";
import { counterStore } from "./counterStore";

const Page = () => {
  return (
    <ul>
      <li>
        <Component />
      </li>
      <li>
        <Component2 />
      </li>
    </ul>
  );
};

export default Page;

const Component = () => {
  const [state, setState] = useStore(counterStore);
  const increment = () =>
    setState((prev) => ({ ...prev, count1: prev.count1 + 1 }));
  const decrement = () =>
    setState((prev) => ({ ...prev, count1: prev.count1 - 1 }));
  const reset = () => setState({ count1: 0, count2: 0 });

  return (
    <div>
      <h1>Count: {state.count1}</h1>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
};

const Component2 = () => {
  const [state, setState] = useStore(counterStore);
  const increment = () =>
    setState((prev) => ({ ...prev, count2: prev.count2 + 1 }));
  const decrement = () =>
    setState((prev) => ({ ...prev, count2: prev.count2 - 1 }));
  const reset = () => setState({ count1: 0, count2: 0 });

  return (
    <div>
      <h1>Count: {state.count2}</h1>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
};
```

다 만들어놓고 보니 굳이 createStore로 한번 더 모듈화를 했어야 하는 고민...
