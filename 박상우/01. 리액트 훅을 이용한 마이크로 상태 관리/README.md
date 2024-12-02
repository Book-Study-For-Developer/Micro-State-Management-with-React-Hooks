### 마이크로 상태 관리 이해하기

리액트 훅 등장 이전까지는 중앙 집중형 상태 관리 라이브러리를 활용하는 것이 일반적이었지만, 상태의 형태에 따라 다양한 상황을 포괄적으로 지원하기에 과한 측면이 있었습니다. ( 폼 데이터 관리, 서버 캐시, 내비게이션 상태 … )

리액트 훅을 통해 다양한 상태를 그 특정한 빙식으로 처리할 수 있게 되었고, 리액트 훅을 기반으로 하는 다양한 상태 관리 라이브러리가 등장하게 되었습니다.

하지만 목적에 맞는 방식으로 모든 문제를 해결할 수는 없었고, 여전히 범용적인 상태 관리가 필요했다.

<u>**범용적인 상태를 관리하는 방법은 가벼워야하고, 요구사항에 따라 적절한 방법을 개발자가 직접 선택할 수 있어야한다는 것**</u>을 마이크로 상태 관리라고 합니다.

### 리액트 훅 사용하기

- 기본 리액트 훅
  - useState
  - useReducer
  - useEffect
- 리액트 훅을 통해서 얻을 수 있는 이점

  - UI 컴포넌트에서 로직 추출 가능

    ```jsx
    // Counter 예시
    // Before
    const Component = () => {
      const [count, setCount] = useState(0);
      return (
        <div>
          {count}
          <button onClick={() => setCount((c) => c + 1)}>+1</button>
        </div>
      );
    };

    // After
    const useCount = () => {
      const [count, setCount] = useState(0);
      const inc = () => setCount((c) => c + 1);
      return [count, inc];
    };

    const Component = () => {
      const [count, inc] = useCount();
      return (
        <div>
          {count}
          <button onClick={inc}>+1</button>
        </div>
      );
    };
    ```

    - 기능 추가에 있어서 컴포넌트 로직을 변경할 필요가 없음

  - 적절한 훅 네이밍을 통해 가독성 향상

- Suspense 와 동시성 랜더링
  - **Suspense**
    - 컴포넌트의 비동기 처리를 담당하는 기능
  - **동시성 랜더링**
    - 랜더링 프로세스를 청크 단위로 분리해서 CPU가 장시간 차단되는 것을 방지하는 방법
- 훅을 활용할 떄 주의할 점
  - state나 ref를 직접 변경하면 안된다.
    → 랜더링 측면에서 부작용이 발생할 수 있음
  - 리액트 훅이나. 컴포넌트 함수는 여러 번 호출 될 수 있는데, 그떄마다 일관되게 동작해야한다. ‘순수’해야 한다.

### 전역상태 탐구하기

- 지역 상태 : 컴포넌트 내에서 useState를 사용하여 정의된 상태
- 전역 상태 : 애플리케이션 내 멀리 떨어진 여러 컴포넌트에서 사용하는 상태. 전역 상태는 싱글 턴일 필요가 없음 ( 공유 상태 )

컴포넌트가 외부 상태에 의존하게 되는 경우 동작이 일관되지 않을 수 있어, 컴포넌트의 주요한 의도인 재사용성에 어려움이 생긴다. 컴포넌트 모델에서는 지역성(Locality)가 중요하며, 컴포넌트가 격리되어야하고 재사용가능한 것이 좋다.

### useState 사용하기

```jsx
const Component = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount(1)}>Set Count to 1</button>
    </div>
  );
};
```

setCount를 통해서 count 값을 1로 바꾸고 컴포넌트가 리랜더링 된다. 하지만 여기서 한번 더 버튼을 클릭하면 setCount를 다시 호출하지만 동일한 값이기 때문에 다시 랜더링되지 않는다. 이것을 **베일아웃(Bailout)** 이라고 한다.

```jsx
// 새로운 객체와 동등비교 하기 때문에 리랜더링 된다.
const Component = () => {
  const [state, setState] = useState({ count: 0 });
  return (
    <div>
      {state.count}
      <button onClick={() => setState({ count: 1 })}>Set Count to 1</button>
    </div>
  );
};

// state 객체가 실제로 변경되지 않았기 때문에 리랜더링 되지 않는다.
const Component = () => {
  const [state, setState] = useState({ count: 0 });
  return (
    <div>
      {state.count}
      <button
        onClick={() => {
          state.count = 1;
          setState(state);
        }}
      >
        Set Count to 1
      </button>
    </div>
  );
};

// 함수 상태로 작성하면 이전 값을 기반으로 갱신이 가능하다.
const Component = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>Increment Count</button>
    </div>
  );
};
```

### useReducer 사용하기

베일 아웃은 useReducer에서도 작동한다.

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 };
    case "SET_TEXT":
      if (!action.text) {
        // 같은 값, state 자체를 넘겨주는 경우에만 bail out
        return state;
      }
      return { ...state, text: action.text };
    default:
      throw new Error("unknown action type");
  }
};
```

```jsx
// delta(action)에 객체가 아니더라도 정상적으로 동작한다.
const reducer = (count, delta) => {
  if (delta < 0) {
    throw new Error("delta cannot be negative");
  }
  if (delta > 10) {
    // too big, just ignore
    return count;
  }
  if (count < 100) {
    // add bonus
    return count + delta + 10;
  }
  return count + delta;
};
```

### useReducer를 이용한 useState 구현

```js
const useState = (initialState) => {
  const [state, dispatch] = useReducer((prev. action) => typeof action === 'function' ? action(prev) : action, intialState);

  return [state, dispatch]
}

// 간소화된 버전
const reducer = (prev, action) => typeof action === 'function' ? action(prev) : action;

const useState = (initialState) => useReducer(reducer, initialState);
```

### useState를 이용한 useReducer 구현

```js
const useReducer = (render, initialState) => {
  const [state, setState] = useState(initialState);
  const dispatch = (action) => setState((prev) => reducer(prev, action));

  return [state, dispatch];
};
```

```js
// 초기화 함수에 대한 구현 추가
const useReducer = (reducer, initalArg, init) => {
  const [state, setState] = useState(
    init ? () => init(initialArg) : initialArg
  );

  const dispatch = useCallback(
    (action) => setState((prev) => reducer(prev, action)),
    [reducer]
  );

  return [state, dispatch];
};
```

<br>

## 추가로 알아본 것

> `리액트 훅은 서스펜스와 동시성 렌더링과 함께 작동하도록 설계 및 개발됐기 때문이다.`라는 문장에서 동시성 렌더링에 대해 제대로 모르고 있어서 공식문서 내용을 살펴보고 넘어갔습니다.

### 1. 동시성 랜더링

**동시성**

둘 이상의 작업이 동시에 진행되는 것. 동시에 두가지 일을 처리하는 것처럼 보이지만, 단일 스레드 환경에서 둘 이상의 작업을 작은 단위로 구분하고 번갈아가며(Context Switching) 마치 동시에 진행되는 것처럼 보이게 처리하는 방식입니다.

<br>

**React에서 동시성**

React에서의 동시성은 여러 버전의 UI를 준비할 수 있는 메커니즘. 랜더링 작업을 작은 단위로 쪼개서 우선순위 기반으로 처리하고, 각 작업은 중단과 재개를 가능하도록해서 여러 작업을 동시에 처리할 수 있게 했습니다.
그리고 동시성 개념을 활용한 Auto-Batching, Suspense, Transition, Streaming Server Rendering과 같은 기능들이 구축되었습니다.

[Concurrent React란 무엇인가요?](https://react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)

### 2. Suspense

Suspense는 비동기 데이터 로딩 중 발생하는 '대기 (Pending) 상태'를 간단하게 처리하기 위해 사용된다. 내부 데이터를 로드하는 훅에서 Promise를 반환하면 Suspense는 이를 감지하고 로딩 상태를 보여줍니다.(Fallback UI)

<br/>
<br/>

> useState와 useReducer에서 지연 초기화를 위해 사용되는 `초기화 함수` 인자가 내부적으로 어떻게 동작하는지 궁금해져서 추가로 React 코드를 뜯어보았습니다.

### 3. 초기화 함수

- useState에서의 초기화 함수

  ```jsx
  function mountState<S>(
    initialState: (() => S) | S
  ): [S, Dispatch<BasicStateAction<S>>] {
    const hook = mountStateImpl(initialState);
    const queue = hook.queue;
    const dispatch: Dispatch<BasicStateAction<S>> = (dispatchSetState.bind(
      null,
      currentlyRenderingFiber,
      queue
    ): any);
    queue.dispatch = dispatch;
    return [hook.memoizedState, dispatch];
  }

  function mountStateImpl<S>(initialState: (() => S) | S): Hook {
    const hook = mountWorkInProgressHook();
    if (typeof initialState === "function") {
      // 초기화 함수가 전달된 경우
      const initialStateInitializer = initialState;
      // $FlowFixMe[incompatible-use]: Flow doesn't like mixed types
      initialState = initialStateInitializer(); // 초기화 함수를 실행하고 그 결과 값을 initalState로 활용
      if (shouldDoubleInvokeUserFnsInHooksDEV) {
        setIsStrictModeForDevtools(true);
        try {
          // $FlowFixMe[incompatible-use]: Flow doesn't like mixed types
          initialStateInitializer(); // strice mode인 경우 초기화 함수를 한번 더 실행해서 부작용을 검사
        } finally {
          setIsStrictModeForDevtools(false);
        }
      }
    }
    // 초기 상태와 업데이트 큐를 포함한 훅을 생성
    hook.memoizedState = hook.baseState = initialState;
    const queue: UpdateQueue<S, BasicStateAction<S>> = {
      pending: null,
      lanes: NoLanes,
      dispatch: null,
      lastRenderedReducer: basicStateReducer,
      lastRenderedState: (initialState: any),
    };
    hook.queue = queue;
    return hook; // 훅을 반환
  }
  ```

- useReducer에서의 초기화 함수

  ```jsx
  function mountReducer<S, I, A>(
    reducer: (S, A) => S,
    initialArg: I,
    init?: (I) => S
  ): [S, Dispatch<A>] {
    const hook = mountWorkInProgressHook();
    let initialState;
    if (init !== undefined) {
      // init 함수가 전달되면 initialArg를 활용하여 초기 상태를 만든다.
      initialState = init(initialArg);
      if (shouldDoubleInvokeUserFnsInHooksDEV) {
        setIsStrictModeForDevtools(true);
        try {
          // strice mode인 경우 초기화 함수를 한번 더 실행해서 부작용을 검사.
          init(initialArg);
        } finally {
          setIsStrictModeForDevtools(false);
        }
      }
    } else {
      initialState = ((initialArg: any): S);
    }
    // 새 훅을 생성하고 초기 상태를 저장.
    hook.memoizedState = hook.baseState = initialState;
    const queue: UpdateQueue<S, A> = {
      pending: null,
      lanes: NoLanes,
      dispatch: null,
      lastRenderedReducer: reducer,
      lastRenderedState: (initialState: any),
    };
    hook.queue = queue;
    const dispatch: Dispatch<A> = (queue.dispatch = (dispatchReducerAction.bind(
      null,
      currentlyRenderingFiber,
      queue
    ): any));
    return [hook.memoizedState, dispatch];
  }
  ```

  ### 4. 궁금해서 더 알아본 부분 : updateState 와 mountState

  setState와 setReducer 코드를 보면서 같은 이름이지만 내부에 `mount~` 를 호출하는 훅과, `update~`를 호출하는 훅이 구분되어 있었습니다.

  ```jsx
    useState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      currentHookNameInDev = 'useState';
      warnInvalidHookAccess();
      mountHookTypesDev(); // mount~ 함수를 호출
      const prevDispatcher = ReactSharedInternals.H;
      ReactSharedInternals.H = InvalidNestedHooksDispatcherOnMountInDEV;
      try {
        return mountState(initialState);
      } finally {
        ReactSharedInternals.H = prevDispatcher;
      }
    },

    useState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      currentHookNameInDev = 'useState';
      warnInvalidHookAccess();
      updateHookTypesDev(); // update~ 함수를 호출
      const prevDispatcher = ReactSharedInternals.H;
      ReactSharedInternals.H = InvalidNestedHooksDispatcherOnUpdateInDEV;
      try {
        return rerenderState(initialState);
      } finally {
        ReactSharedInternals.H = prevDispatcher;
      }
    },
  ```

  같은 이름의 훅을 내부적으로 어떻게 구분하고 호출하는지 궁금해져서 조금 더 알아보았습니다.

  useState가 호출되었을 때,

  ```jsx
  export function useState<S>(
    initialState: (() => S) | S
  ): [S, Dispatch<BasicStateAction<S>>] {
    const dispatcher = resolveDispatcher();
    return dispatcher.useState(initialState);
  }
  ```

  `resolveDispatcher`를 호출하여 dispatcher를 결정합니다.

  ```jsx
  function resolveDispatcher() {
    const dispatcher = ReactSharedInternals.H;
    if (__DEV__) {
      if (dispatcher === null) {
        console.error(
          "Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for" +
            " one of the following reasons:\n" +
            "1. You might have mismatching versions of React and the renderer (such as React DOM)\n" +
            "2. You might be breaking the Rules of Hooks\n" +
            "3. You might have more than one copy of React in the same app\n" +
            "See https://react.dev/link/invalid-hook-call for tips about how to debug and fix this problem."
        );
      }
    }
    // Will result in a null access error if accessed outside render phase. We
    // intentionally don't throw our own error because this is in a hot path.
    // Also helps ensure this is inlined.
    return ((dispatcher: any): Dispatcher);
  }
  ```

  `resolveDispatcher`는 `ReactSharedInternals.H`를 통해 현재 사용하는 디스패처를 저장하고 반환하는 역할을 합니다.

  `ReactSharedInternals`는 React 내에서 공용으로 사용되는 설정들을 활용하기 위한 값이라고 합니다. `ReactSharedInternals`내부에서 활성화된 디스패처에 따라 어떤 훅이 사용될지 결정되는 것 같습니다.

  update 될 때는 `HooksDispatcherOnUpdate`를, mount 될 때는 `HooksDispatcherOnMount`를 활용함으로서 각 단계에 따라 다르게 동작시킬 수 있습니다.

  ```jsx
  const HooksDispatcherOnUpdate: Dispatcher = {
    ...
    useReducer: updateReducer,
    useState: updateState,
    ...
  };
  ```

  ```jsx
  const HooksDispatcherOnMount: Dispatcher = {
    ...
    useReducer: mountReducer,
    useState: mountState,
    ...
  };
  ```

  그런데 코드를 따라가면서 제가 궁금증을 가졌던 코드인 같은 이름의 useState을 만날 수는 없었습니다.
  처음에 제시했던 [두 코드](#4-궁금해서-더-알아본-부분--updatestate-와-mountstate)는 `HooksDispatcherOnUpdateInDEV`와 `HooksDispatcherOnMountInDEV`의 메소드 들이었습니다.

  `~InDEV`라는 postfix가 붙은 함수들의 왜 필요한지 마지막으로 알아보았습니다.

  `~InDEV`로 정의된 객체들은 개발 모드에서 기본적인 동작과 더불어 Hook 규칙, 훅이 중첩으로 사용되었는지 검증과 디버깅에 활용하기 위해 React DevTools와 연동 하기 위해 만들어졌다고 합니다.

  ```jsx
  export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
  ): any {
    ...
    if (__DEV__) { // DEV 환경에서는 ~InDEV 디스패처가 활용됩니다.
      if (current !== null && current.memoizedState !== null) {
        ReactSharedInternals.H = HooksDispatcherOnUpdateInDEV;
      } else if (hookTypesDev !== null) {
        // This dispatcher handles an edge case where a component is updating,
        // but no stateful hooks have been used.
        // We want to match the production code behavior (which will use HooksDispatcherOnMount),
        // but with the extra DEV validation to ensure hooks ordering hasn't changed.
        // This dispatcher does that.
        ReactSharedInternals.H = HooksDispatcherOnMountWithHookTypesInDEV;
      } else {
        ReactSharedInternals.H = HooksDispatcherOnMountInDEV;
      }
    } else {
      ReactSharedInternals.H =
        current === null || current.memoizedState === null
          ? HooksDispatcherOnMount
          : HooksDispatcherOnUpdate;
    }
    ...
    return children;
  }
  ```
