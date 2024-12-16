> 리액트 컴포넌트에서 모듈 상태를 사용하기 위해서는 구독(subscription)을 사용해야 한다.

### 모듈 상태 살펴보기

```tsx
export const createContainer = initialState => {
  let state = initialState
  const getState = () => state
  const setStatae = nextState => {
    state = typeof nextState === 'function' ? nextState(state) : nextState
  }
  return { getState, setState }
}
```

### 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

- 전체 트리에서 전역 상태가 필요한 경우 모듈 상태가 적합
- 하지만 리액트 컴포넌트에서 모듈 상태를 사용하려면 리렌더링 최적화를 직접 처리해야 함
- 리액트에서 리렌더링을 일으키는 훅은 `useState`와 `useReducer`로 모듈 상태에 반응하려면 반드시 둘 중 하나를 사용해야 함
- 또한, 컴포넌트 생명 주기를 고려하면서 컴포넌트 외부에 있는 모듈 수준에서 `setState`를 관리할 필요성
  - `useEffect` 훅 + `setState`를 관리하는 별도의 Set 자료구조

### 기본적인 구독 추가하기

- 구독: 갱신(update)에 대한 알림을 받기 위한 방법

```tsx
const unsubscribe = store.subscribe(() => {
  console.log('store is updated')
})
```

#### Store 생성하기

- 초기 상태 값을 통해 `store`를 생성하는 `createStore` 함수

```tsx
type Store<T> = {
  getState: () => T
  setState: (action: T | ((prev: T) => T)) => void
  subscribe: (callback: () => void) => () => void
}

const createStore = <T extends unknown>(initialState: T): Store<T> => {
  let state = initialState
  const callbacks = new Set<() => void>() // 구독 관리용 자료구조(중복 방지를 위해 Set 사용)
  const getState = () => state // 상태 조회
  const setState = (nextState: T | ((prev: T) => T)) => {
    // 상태 변경
    state = typeof nextState === 'function' ? (nextState as (prev: T) => T)(state) : nextState
    callbacks.forEach(callback => callback())
  }
  const subscribe = (callback: () => void) => {
    // 구독 관리
    callbacks.add(callback)
    return () => {
      callbacks.delete(callback) // 구독 해제 함수 반환
    }
  }

  return { getState, setState, subscribe }
}
```

- `createStore` 사용하기

```tsx
const store = createStore({ count: 0 })
```

#### 커스텀 훅 `useStore`

- store를 받아서 store의 상태 값과 갱신 함수를 튜플로 반환하는 커스텀 훅
- `useState + useEffect` 조합

```tsx
const useStore = <T extends unknown>(store: Store<T>) => {
  const [state, setState] = useState(store.getState())
  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(store.getState())
    })
    setState(store.getState()) // tearing 방지??
    return unsubscribe
  }, [store])
  return [state, store.setState] as const
}
```

> 모듈 상태는 결국 리액트에서 갱신되기 때문에 **리액트의 상태와 동일하게 모듈 상태를 불변적으로 갱신하는 것**이 중요하다.

### 상태 범위를 지정해서 사용하는 `useStoreSelector`

- 선택자 함수 selector를 받아서 상태 범위를 지정

```tsx
const useStoreSelector = <T, S>(store: Store<T>, selector: (state: T) => S) => {
  const [state, setState] = useState(() => selector(store.getState()))
  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      setState(selector(store.getState()))
    })
    setState(selector(store.getState()))
    return unsubscribe
  }, [store, selector])
  return state
}
```

- `useStore`, `useStoreSelector` 훅 사용 시 주의 사항
  - `useEffect`는 조금 늦게 실행되기 때문에 재구독될 때까지는 갱신되기 이전 상태 값 반환 → **tearing**
  - 이를 해결하기 위해 리액트에서 `useSubscription`을 계승한 `useSyncExternalStore`를 제공하고 있음

> [React 18 for External Store Libraries](https://youtu.be/oPfSC5bQPR8?si=fZzigaPWVBbXPkaL)
>
> → 저자가 소개하는 `useSyncExternalStore` 영상인데 해당 챕터에서 소개하는 예제 코드가 나온다..
>
> → `useSyncExternalStore` 훅을 처음 봤을 때 좀 생소했는데, 결국 `useState + useEffect`로 외부 상태를 동기화 및 관리하던 걸 티어링이나 정확한 업데이트 보장이 어려운 점 등을 보완하여 좀 더 안정적으로 동기화할 수 있도록 리액트 팀에서 제공해준 훅이라고 생각하는 이해가 좀 수월했다.
