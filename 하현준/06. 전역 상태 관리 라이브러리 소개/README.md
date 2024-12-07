## 전역 상태 관리 문제 해결하기

**문제점**

- 전역 상태를 읽는 방법
  - 전역 상태의 값이 객체로 이뤄졌을 경우 변경된 값이 컴포넌트에 연관이 없어도 리렌더링 되는 케이스를 고려해야 한다.
- 전역 상태에 값을 넣거나 갱신하는 방법
  - 중첩된 객체가 있을 경우에 직접적으로 변경하면 안된다. (불변성!)

### 전역 상태 관리 vs 대 범용 상태 관리

- 범용 상태관리를 위해 자주 사용되는 것은 Redux(단방향 데이터 흐름) 또는 XState(상태 머신 기반)을 사용

**XState 알아보기**

> XState is a state management and orchestration solution for JavaScript and TypeScript apps.
> It uses [event-driven](https://stately.ai/docs/transitions) programming, [state machines, statecharts](https://stately.ai/docs/state-machines-and-statecharts), and the [actor model](https://stately.ai/docs/actor-model) to handle complex logic in predictable, robust, and visual ways. XState provides a powerful and flexible way to manage application and workflow state by allowing developers to model logic as actors and state machines. It integrates well with React, Vue, Svelte, and other frameworks and can be used in the frontend, backend, or wherever JavaScript runs.

XState는 JavaScript 및 TypeScript 앱을 위한 상태 관리 및 조정 솔루션입니다.
이벤트 중심 프로그래밍, 상태 머신, 상태 차트, 행위자 모델을 사용하여 예측 가능하고 강력하며 시각적인 방식으로 복잡한 논리를 처리합니다. XState는 개발자가 로직을 행위자 및 상태 시스템으로 모델링할 수 있도록 하여 애플리케이션 및 워크플로 상태를 관리하는 강력하고 유연한 방법을 제공합니다. React, Vue, Svelte 및 기타 프레임워크와 잘 통합되며 프런트엔드, 백엔드 또는 JavaScript가 실행되는 모든 곳에서 사용할 수 있습니다.

**유한 상태 기계(Finite State Machine)의 개념**

- FSM은 유한 상태 기계를 나타내는 디자인 패턴입니다
- 상태와 상태 간의 전환을 기반으로 동작하는 동작 기반 시스템입니다.

**FSM의 구성 요소**

- 상태 (State): 시스템이 취할 수 있는 다양한 상태를 나타냅니다.
- 전환 조건 (Transition Condition): 상태 간 전환을 결정하는 조건입니다.
- 동작 (Action): 상태에 따라 수행되는 동작 또는 로직을 나타냅니다.

사용 예시)

- https://stately.ai/blog/2024-02-12-xstate-react-global-state
- https://fe-developers.kakaoent.com/2022/220922-make-cart-with-xstate/

## 데이터 중심 접근 방식과 컴포넌트 중심 접근 방식

**데이터 중심 접근 방식이란?**

- 데이터 모델을 리액트 **외부에 설계한 뒤에 컴포넌트와 연결**하는 방식이다.
- 컴포넌트 외부(메모리)에 존재하기 때문에 리액트 **컴포넌트 렌더링 전** 또는 **마운트 해제된 후**에도 존재할 수 있다.
- 5장까지 작업한 내용이 이에 해당한다고 볼 수 있다. (useSyncExternalStore, context + subscribe)

**컴포넌트 중심 접근 방식이란?**

- 컴포넌트 중심적으로 접근하기 때문에 컴포넌트 라이프 사이클 내에 전역 상태를 유지하는 것
- **컴포넌트 렌더링 전** 에는 존재할 수 없고, **마운트 해제된 후**에는 같이 사라지게 된다.

## 리렌더링 최적화

위 문제점에서 얘기했듯이 불필요한 리렌더링이 일어나는 것을 피해야하는데 이를 위해 세가지 방법을 소개하고 있다.

- 선택자 함수 사용
- 속성 접근 감지
- 아톰 사용

### 선택자 함수 사용

지금까지 계속 했던 useSelector를 사용하는 방법이다.

```tsx
const Component = () => {
  const value = useSelector((state) => state.b.c);

  return <>{value}</>;
};
```

여태까지는 계속 원시값에 대해서만 얘기하였지만 객체를 리턴하게 될 경우 메모이제이션을 사용해야 한다.

코드로 예를 들자면,,

```tsx
const Component = () => {
  const user = useSelector((state) => state.user);
  // user가 객체일 경우에는 useMemo로 감싸서 바뀌지 않았는지 체크하는 방식이 포함되어야 한다.
  const memoedUser = useMemo(() => user, [user]);

  return <>{value}</>;
};
```

### 속성 접근 감지

selector로 직접 명시하지 않고 자동으로 추적할 수 있는 방식이 있다. 바로 **속성 접근을 감지하고 감지한 정보를 렌더링 최적화에 사용할 수 있는 상태 사용 추적(state usage tracking)**을 사용하면 된다.

책에서는 슈도 코드로 작성해서 알려주는데 뒤에 소개하는 라이브러리에서 사용되는 방법 중 하나로 소개될 것 같다. 이를 자동으로 탐지하기 위해서는 proxy를 통해 확인하면 된다.

→ `vue`에서 상태들이 proxy로 구현되어 있어서 `vue`에서는 자동으로 최적화를 해주는 방법이 녹아져 있다.

MDN: proxy ⇒ https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Proxy

`useSelector`와 `useTrackedState`와의 차이점

```tsx
// selector 사용한 예시
const Component = () => {
  const isSmall = useSelector((state) => state.a < 10);
  return <>{isSmall ? "small" : "big"}</>;
};

// useTrackedState 사용한 예시
const Component = () => {
  const isSmall = useTrackedState().a < 10;
  return <>{isSmall ? "small" : "big"}</>;
};
```

`useTrackedState` 는 a가 변할 때마다 리렌더링 되지만, `useSelector`는 isSmall이 바뀔 때만 리렌더링 된다. 이를 쉽게 풀어서 얘기하자면,

- a의 변화
  - 5 → 3 → 2→ 11
  - `useSelector` 는 isSmall이 변하는게 한 번(2 → 11)이므로 한번만 리렌더링 된다.
  - 반면에 `useTrackedState` 는 a가 4번이 변했기 때문에 4번 리렌더링 되게 되는 것이다.

### 아톰 사용

리렌더링을 발생시키는 데 사용되는 최소 상태 단위로 세분화해서 구독을 하여 사용하는 방식이다.

```tsx
const globalState = {
  a: atom(1),
  b: atom(2),
  c: atom(3),
};

const Component = () => {
  const value = useAtom(globalState.a);
  return <>{value}</>;
};
```

jotai가 이러한 방식을 채택했는데 이는 수동 최적화와 자동 최적화의 중간 정도로 볼 수 있다.

- 수동 최적화: globalState에 정의한 a, b, c값들
- 자동 최적화: 컴포넌트에서 사용되고 있는 value (value가 변하게 되면 의존성 추적해 자동으로 최적화 해줌)
