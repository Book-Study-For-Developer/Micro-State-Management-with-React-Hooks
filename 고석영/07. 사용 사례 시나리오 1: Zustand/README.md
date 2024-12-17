> [!NOTE]
>
> ### Zustand
>
> - 주로 리액트의 **모듈 상태(`store`)**를 생성하도록 설계된 작은 라이브러리
> - 상태 객체를 수정할 수 없고 항상 새로 만들어야 하는 **불변 갱신 모델**이 기반
> - 선택자를 사용한 **수동 렌더링 최적화**

## 모듈 상태와 불변 상태 이해하기

> Zustand는 주로 모듈 상태를 위해 설계됐으므로 모듈에서 store를 정의하고 내보내는 것을 할 수 있다.

### 불변 상태 모델 기반

- 상태 변경 시 새 객체를 생성해서 대체하고, 수정하지 않은 객체는 재사용

```tsx
const state1 = store.getState()
state1.count = 2 // ❌ 새로운 상태가 이전 상태와 동일한 참조를 가짐 → 변경 사항 감지 X
store.setState(state1)
```

#### 불변 상태 모델의 장점

- 상태 객체의 참조에 대한 동등성만 확인하면 변경 여부를 알 수 있으므로 객체의 값 전체를 확인할 필요가 없음

```tsx
import create from 'zustand'

export const store = create(() => ({ count: 0 }))
```

#### `create` 함수

- Zustand에서 상태를 정의하고 관리하는 함수

```tsx
create<T>()(stateCreatorFn: StateCreator<T, [], []>): UseBoundStore<StoreApi<T>>
```

- `create` 함수 인자
  - `stateCreatorFn`: 상태를 정의하는 함수
    - `set`: 상태 갱신 함수
    - `get`: 현재 상태 반환 함수
    - `store`: 전체 stor API 객체
- `create` 함수 반환값
  - `getState`: store 상태 읽기
  - `setState`: store 상태 갱신 → 값, 함수 갱신 모두 가능
  - `subscribe`: store 상태 구독
  - `getInitialState`: store 초기 상태 반환 (SSR 고려)
- `setState`
- 값, 함수 갱신 모두 지원
- setState는 `Object.assign` 으로 구현되어 새 상태와 이전 상태를 병합하므로, 설정하려는 속성만 지정도 가능

```tsx
// https://github.com/pmndrs/zustand/blob/main/src/vanilla.ts
const setState: StoreApi<TState>['setState'] = (partial, replace) => {
  const nextState =
    typeof partial === 'function' ? (partial as (state: TState) => TState)(state) : partial

  if (!Object.is(nextState, state)) {
    const previousState = state
    state =
      replace || typeof nextState !== 'object' || nextState === null
        ? (nextState as TState)
        : Object.assign({}, state, nextState)

    listeners.forEach(listener => listener(state, previousState))
  }
}
```

- 비교 알고리즘: 상태의 동일성 판단 → `Object.is`
- 병합 알고리즘: 객체 병합을 통한 상태 갱신 → `Object.assign`

## 리액트 훅을 이용한 리렌더링 최적화

> 전역 상태를 사용하는 경우 모든 컴포넌트가 전역 상태를 사용하는 것은 아니기 때문에 리렌더링 최적화가 필요하다.

- 리액트에서 store를 사용하려면 사용자 정의 훅이 필요
  - 리액트와의 연결을 위해 `useSyncExternalStore` 활용

```tsx
// https://github.com/pmndrs/zustand/blob/main/src/react.ts
export function useStore<TState, StateSlice>(
  api: ReadonlyStoreApi<TState>,
  selector: (state: TState) => StateSlice = identity as any,
) {
  const slice = React.useSyncExternalStore(
    // 리액트와의 동기화
    api.subscribe,
    () => selector(api.getState()),
    () => selector(api.getInitialState()),
  )
  React.useDebugValue(slice) // React-Devtools 디버깅용 라벨 추가
  return slice
}
```

- `create` 함수: 훅으로 사용할 수 있는 store를 생성 → 이때, 훅은 리액트 규칙을 따라 `useStore`로 명명

```tsx
const useStore = create(() => ({
  count: 0,
  text: 'hello',
}))
```

- 리렌더링을 피해야 하는 경우 `useStore`에 선택자 함수 지정

```tsx
const count = useStore(state => state.count)
```

- 직접 명시 → 선택자 기반 리렌더링 제어를 수동 렌더링 최적화라고 함
- 리렌더링을 피하기 위해 선택자는 선택자 함수가 반환하는 결과를 비교하는 방식으로 작동

```tsx
const Component = () => {
  const [{ count }] = useStore(state => [{ count: state.count }])
  return <div>count: {count}</div>
}
```

- count 값이 변경되지 않은 경우에도 컴포넌트가 리렌더링
  - 위 코드에서 `[{ count: state.count }]`를 **배열로 래핑**
  - 배열이나 객체는 **참조값**을 비교하기 때문에 매 렌더링마다 `[{ count: state.count }]`라는 새로운 배열을 생성
  - 즉, 리액트는 이 배열의 참조가 바뀌었다고 판단 → 리렌더링
- 선택자 기반 렌더링 최적화의 장점과 단점
  - 장점: 선택자 함수를 명시적으로 작성 → 정확한 동작 예측
  - 단점: 객체 참조에 대한 이해

## 읽기 상태와 갱신 상태 사용하기

> Zustand는 다양한 방식으로 사용할 수 있는 라이브러리지만, 상태를 읽고 갱신하는 몇 가지 패턴이 있다.

### 선택자 함수 미리 정의하기

```tsx
const selectCount1 = (state: StoreState) => state.count1
const selectCount2 = (state: StoreState) => state.count2
```

- 선택자 함수를 미리 만든 후 전달하면 리렌더링 최적화 가능
- 더 높은 재사용성과 가독성을 위해 `store`에 함수를 미리 정의할 수도 있음 → `set` 함수 사용
  - 상태 로직이 컴포넌트 외부에 위치 가능

### 파생 상태(Derived State)

#### 파생 상태가 필요한 상황

```tsx
const Total = () => {
  const count1 = useStore(selectCount1)
  const count2 = useStore(selectCount2)
  return <div>total: {count1 + count2}</div>
}
```

- 상태를 계산해서 얻어지는 값이 필요한 경우 파생 상태 활용 가능
- 현재 상태에선 `count1 + count2` 의 결과가 동일해도 리렌더링 발생 → 리렌더링이 발생하지 않게 파생 상태를 사용하려면?

#### 방법 1: 파생 상태를 선택자로 계산해서 넘기기

```tsx
const selectTotal = (state: StoreState) => state.count1 + state.count2
```

#### 방법2: store에서 파생 상태를 생성하기

- 결과를 기억할 수 있고 많은 컴포넌트가 값을 사용할 때 불필요한 계산을 피할 수 있음

```tsx
const useStore = create(set => ({
  count1: 0,
  count2: 0,
  total: 0,
  inc1: () =>
    set(prev => ({
      ...prev,
      count1: prev.count1 + 1,
      total: prev.count1 + 1 + prev.count2,
    })),
  inc2: () =>
    set(prev => ({
      ...prev,
      count2: prev.count2 + 1,
      total: prev.count2 + 1 + prev.count1,
    })),
}))
```

## 선택자 기반 접근 방식의 Zustand 장단점

### Zustand의 읽기 및 쓰기 상태

- 읽기 상태: 리렌더링을 최적화하기 위해 선택자 함수 사용
- 쓰기 상태: 불변 상태 모델을 기반으로 함

#### 장점

- 리액트의 객체 불변성 규칙과 일치하는 접근 방식
  - Zustand의 상태 모델은 리액트의 불변성 규칙과 일치
  - 선택자 함수를 사용한 렌더링 최적화 역시 불변성을 기반으로 함
- 라이브러리의 단순성과 번들 크기가 작음 → v5.0.2 기준 588B

|                                    | zustand | jotai   | valtio    | react-tracked |
| ---------------------------------- | ------- | ------- | --------- | ------------- |
| 버전                               | v5.0.2  | v2.10.3 | v2.1.2    | v2.0.1        |
| 번들 사이즈 (Minified + Gzip 압축) | 588B    | 3.5kB   | **2.7kB** | 2.1kB         |

> 번들 사이즈 비교는 [bundlephobia](https://bundlephobia.com)를 참고했는데요. v5 버전이 바뀌면서 번들 사이즈가 절반 이상 줄어들었네요 ✨ 어떤 마법이 있었는지 궁금해서 릴리즈 노트를 봤으나 별다른 소득은 없었던.. 혹시 이유를 아시는 분 있으신가요?
>
> ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/fc48ee3d-1b55-4437-92f8-fdb7b54dff50/19ec7ea1-9f42-4d9f-8784-e4be7d022428/image.png)
>
> - https://bundlephobia.com/package/zustand@4.5.4
> - [Zustand Release Note (v5.0.0)](https://github.com/pmndrs/zustand/releases/tag/v5.0.0)
> - [Announcing Zustand v5](https://pmnd.rs/blog/announcing-zustand-v5)

#### 단점

- 선택자를 이용한 수동 렌더링 최적화
- 객체 참조 동등성을 이해해야 하며, 선택자 코드를 위해 보일러 플레이트 코드를 많이 작성해야 할 필요가 있음
