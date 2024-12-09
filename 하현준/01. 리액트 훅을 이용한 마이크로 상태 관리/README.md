## 마이크로 상태 관리 이해하기

**책에서 설명하는 마이크로 상태 관리**

- 상태 관리를 위한 방법은 가벼워야 하고 이를 마이크로 상태 관리라고 한다.

**내가 보고 느낀 마이크로 상태 관리**

- SRP 원칙을 지키면서 상태를 나누는 느낌이 아닐까 했다.
  - 단일 책임으로 상태를 묶으면 결국 가벼운 형태로 묶일 것인데, 여기서 가볍다는 개수가 적다라는 느낌보다는 책무가 같은 것 끼리 묶는 것에 가까운 느낌..

책에서 예시를 든 useCount와 같은 것도 결국 Count라는 책무로 상태를 묶었기 때문이다.

초기 설계를 useCount로 시작해 나중에 점차 추가되는 기능에도 대응이 가능해지는 역할도 해준다.

## 비동시성 렌더링 vs 동시성 렌더링

**동시성(Concurrency)을 먼저 알아보자.**

- **한번에 둘 이상의 작업이 동시에 진행되는 것**을 의미
- 리액트 18이전에는 렌더링은 동기적인 처리로 이뤄졌기 때문에 **렌더링이 오래 걸리면 다음 작업이 블로킹** 되는 현상이 있었다.
- 이를 통해 일정 시간을 대기하지 않고 동시에 작업을 수행할 수 있게 되었다.

참고: [React의 Concurrent이란?](https://ko.react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)

그렇다면 비동시성 렌더링은 반대로 **동기적으로 작업이 이뤄지는 것을 의미한다.**

그렇기 때문에 함수가 여러번 호출되어 순수하지 않더라도 문제가 없는 것 처럼 보이게 된다.

## useState에 대하여

```tsx
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

여기서 한번 더 클릭하면 ‘베일아웃’ 되어 렌더링되지 않는다고 얘기하는데, **베일아웃(bailout)이란 리렌더링을 발생시키지 않는 것을 의미**한다.

> 리액트에서 베일아웃을 하는 이유

불변성과 관련되어 변하지 않는 값에 대해서는 성능을 최적화하기 위해 리렌더링하지 않는 것

재조정 단계에서 베일아웃이 일어나는 과정을 다 자세히 알고 싶다면 → https://ted-projects.com/react-internals-deep-dive-13

>

이후 useState의 여러 사용법에 따른 예시를 보여주는데 이는 예시 코드로 확인하기 → https://github.com/wikibook/msmrh/blob/main/chapter01/03_usestate.js

## useReducer에 대하여

useReducer도 useState과 마찬가지로 베일아웃은 이뤄짐.

비슷하게 useReducer의 사용법은 흔히 알고 있는 방식이고 베일아웃이 이뤄지기 위해서는 정확히 동일한 것을 리턴할 때만 이뤄지게 된다.

```tsx
const reducer = (state, action) => {
  switch (action.type) {
    //...
    case 'SET_TEXT':
      if (!action.text) {
        // bail out
        return state;
      }
      // 다른 객체이므로 참조가 달라짐
      return { ...state, text: action.text };
    default:
      throw new Error('unknown action type');
  }
};
```

## useReducer를 활용한 useState 구현

❓: 실제 리액트 내부에서 useReducer를 이용해 구현되어 있다고 하는데 ~~관련 코드를 못찾아서.. 찾으신분이 있다면 알려주세요!~~ → 정리하는 도중에 찾았습니다. ㅋㅋ

https://github.com/facebook/react/blob/main/packages/react/index.js#L72 ⇒ https://github.com/facebook/react/blob/main/packages/react/src/ReactClient.js#L54 ⇒

https://github.com/facebook/react/blob/main/packages/react/src/ReactHooks.js#L96C1-L101C2

실제로 내부를 뜯어보니..

```tsx
export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

export function useReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useReducer(reducer, initialArg, init);
}
```

그럼 `resolveDispatcher` 는 무엇인가?

```tsx
function resolveDispatcher() {
  const dispatcher = ReactSharedInternals.H;
  if (__DEV__) {
    if (dispatcher === null) {
      console.error(
        'Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for' +
          ' one of the following reasons:\n' +
          '1. You might have mismatching versions of React and the renderer (such as React DOM)\n' +
          '2. You might be breaking the Rules of Hooks\n' +
          '3. You might have more than one copy of React in the same app\n' +
          'See https://react.dev/link/invalid-hook-call for tips about how to debug and fix this problem.',
      );
    }
  }
  // Will result in a null access error if accessed outside render phase. We
  // intentionally don't throw our own error because this is in a hot path.
  // Also helps ensure this is inlined.
  return ((dispatcher: any): Dispatcher);
}
```

`ReactSharedInternals` 는 무엇인지..

https://github.com/facebook/react/blob/main/packages/react/src/ReactHooks.js#L19 ⇒

https://github.com/facebook/react/blob/main/packages/shared/ReactSharedInternals.js#L12-L15

```tsx
import * as React from 'react';

const ReactSharedInternals =
  React.__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE;

export default ReactSharedInternals;
```

더 파고 들어가보자..

https://github.com/facebook/react/blob/main/packages/react/src/ReactSharedInternalsClient.js#L43-L60

```tsx
const ReactSharedInternals: SharedStateClient = ({
  H: null,
  A: null,
  T: null,
  S: null,
}: any);

// SharedStateClient란 ???
// https://github.com/facebook/react/blob/main/packages/react/src/ReactSharedInternalsClient.js#L14

import type {Dispatcher} from 'react-reconciler/src/ReactInternalTypes';

export type SharedStateClient = {
  H: null | Dispatcher, // ReactCurrentDispatcher for Hooks
  A: null | AsyncDispatcher, // ReactCurrentCache for Cache
  T: null | BatchConfigTransition, // ReactCurrentBatchConfig for Transitions
  S: null | ((BatchConfigTransition, mixed) => void), // onStartTransitionFinish

  // DEV-only

  // ReactCurrentActQueue
  actQueue: null | Array<RendererTask>,

  // Used to reproduce behavior of `batchedUpdates` in legacy mode.
  isBatchingLegacy: boolean,
  didScheduleLegacyUpdate: boolean,

  // Tracks whether something called `use` during the current batch of work.
  // Determines whether we should yield to microtasks to unwrap already resolved
  // promises without suspending.
  didUsePromise: boolean,

  // Track first uncaught error within this act
  thrownErrors: Array<mixed>,

  // ReactDebugCurrentFrame
  getCurrentStack: null | (() => string),
};
```

그럼 이제 `Dispatcher`가 뭔지 보면 된다.

https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactInternalTypes.js#L396-L405

```tsx
export type Dispatcher = {
  //...
  useState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>];
  useReducer<S, I, A>(
    reducer: (S, A) => S,
    initialArg: I,
    init?: (I) => S,
  ): [S, Dispatch<A>];
  //...
};
```

여기서 useState는 Dispatcher를 사용해 구현되어 있었고, useReducer는 없었다.

결국 useState를 사용하는 곳을 봐야 하는데, 이는 여기있다.

https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L4069

```tsx
{

  useState: mountState,

}
```

https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L4308-L4320

```tsx
 useState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      currentHookNameInDev = 'useState';
      mountHookTypesDev();
      const prevDispatcher = ReactSharedInternals.H;
      ReactSharedInternals.H = InvalidNestedHooksDispatcherOnMountInDEV;
      try {
        return mountState(initialState);
      } finally {
        ReactSharedInternals.H = prevDispatcher;
      }
    },
```

https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L1973C1-L1985C2

```tsx

function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountStateImpl(initialState);
  const queue = hook.queue;
  const dispatch: Dispatch<BasicStateAction<S>> = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any);
  queue.dispatch = dispatch;
  return [hook.memoizedState, dispatch];
}
```

여기서 더 파고 들면 리액트 내부 동작까지 다 파고 들어야해서, 일단 최종적으로 return하는 두개의 배열은 찾았으니 멈췄다. 즉 결국 실제 구현체는 리액트의 Fiber 트리와 엮여서 동작하는 것.

그럼 useReducer로 구현했던 건 어디에?

책에서 말한 useReducer를 사용한 흔적은.. 여기 남아있었다.

https://github.com/facebook/react/blob/c11c9510fa14bbd87053685c19bfdfec2f427f49/packages/react-server/src/ReactFizzHooks.js#L348-L359

```tsx
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  // $FlowFixMe[incompatible-use]: Flow doesn't like mixed types
  return typeof action === 'function' ? action(state) : action;
}

export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  if (__DEV__) {
    currentHookNameInDev = 'useState';
  }
  return useReducer(
    basicStateReducer,
    // useReducer has a special case to support lazy useState initializers
    (initialState: any),
  );
}
```

그럼 이거와 위는 무엇의 차이인지..? 커밋 기록을 봅시다

https://github.com/facebook/react/pull/21257

❓: Fizz는 Fiber와 다른 방식의 렌더러..? 를 구현한 느낌인데 이 부분 알게 되면 서로 공유해줘요..!

- 제가 살펴본 결과 `react-server` 패키지 내부에 들어있는데 이는 서버 사이드를 위한 새로운 렌더링 방식이 아닐까 싶습니다.
  - This is an experimental package for creating custom React streaming server renderers.
    리액트 서버 렌더러를 커스텀하게 만들수 있는 실험적인 패키지라고 소개..

https://github.com/facebook/react/tree/main/packages/react-server

> `Fizz` is a renderer for Server Side Rendering React. The same code that runs in the client (browser or native) is run on the server to produce an initial view to send to the client before it has to download and run React and all the user code to produce that view on the client.

### 그럼 useReducer는??

https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js#L4500

```tsx
 useReducer<S, I, A>(
      reducer: (S, A) => S,
      initialArg: I,
      init?: I => S,
    ): [S, Dispatch<A>] {
      currentHookNameInDev = 'useReducer';
      updateHookTypesDev();
      const prevDispatcher = ReactSharedInternals.H;
      ReactSharedInternals.H = InvalidNestedHooksDispatcherOnMountInDEV;
      try {
        return mountReducer(reducer, initialArg, init);
      } finally {
        ReactSharedInternals.H = prevDispatcher;
      }
    },
```

비슷하게 내부는 또 mountReducer를 사용중이고, mountReducer도 트리와 엮인 상태로 구현이 되어 있다.

```tsx
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
    if (shouldDoubleInvokeUserFnsInHooksDEV) {
      setIsStrictModeForDevtools(true);
      try {
        init(initialArg);
      } finally {
        setIsStrictModeForDevtools(false);
      }
    }
  } else {
    initialState = ((initialArg: any): S);
  }
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
    queue,
  ): any));

  // 리턴하는 두개의 인자
  return [hook.memoizedState, dispatch];
}
```

아까 위에서 언급한 react-server의 useReducer는 구현체에 직접 workInProgressHook이 들어가있다.

코드가 너무 커서 직접 보는게 나을 듯 해서 링크로 남긴다.

https://github.com/facebook/react/blob/c11c9510fa14bbd87053685c19bfdfec2f427f49/packages/react-server/src/ReactFizzHooks.js#L361

## useState와 useReducer의 사소한 차이점

- 초기화 함수 사용할 때 외부에 변수를 둘 수 있냐 없냐의 차이
- useRedcuer에서만 인라인 리듀서를 사용할 수 있음.
  ```tsx
  const useScore = (bonus) =>
    useReducer((prev, delta) => prev + delta + bonus, 0);
  ```
  - ❓: 이것을 쓸일이 있을까요?? 한번도 써보지 않은 형태네요
