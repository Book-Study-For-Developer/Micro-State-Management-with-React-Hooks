### 컨텍스트+구독

- 컨텍스트의 이점
  - 하위 트리에 전역 상태 제공 가능
  - 컨텍스트 공급자 중첩 가능
  - **리액트 컴포넌트 생명 주기 내에서 `useState` 등으로 전역 상태 제어 가능**
- 구독의 이점
  - 단일 컨텍스트로는 불가능한 리렌더링 문제 해결 가능

### 모듈 상태의 한계

> 모듈 상태는 리액트 컴포넌트 외부에 존재하는 전역으로 정의된 **싱글턴**이기 때문에 **컴포넌트 트리나 하위트리마다 다른 상태를 가질 수 없다**는 한계가 있다.

- 모듈 상태로 전역 상태인 store 생성

```tsx
const store = createStore({ count: 0 })
```

- 여기서 store는 리액트 컴포넌트 외부에 정의되어 있음 → 리액트로의 연결이 필요 → `useStore` 사용

```tsx
const Counter = () => {
  const [state, setState] = useStore(store)
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }))
  }

  return (
    <div>
      {state.count} <button onClick={inc}>+1</button>
    </div>
  )
}

const Component = () => (
  <>
    <Counter />
    <Counter />
  </>
)
```

> 여기에서 Component에 다른 한 쌍의 Counter 컴포넌트를 추가하고 싶지만, 새로운 한 쌍은 첫 번째와 카운트를 보여줘야 한다면?

- 새로운 모듈 상태(store2)를 만들고, 해당 모듈 상태를 사용할 컴포넌트(Counter2)를 새로 만들어야 함
- 재사용 할 수 없나요? → 모듈 상태는 리액트 외부에서 정의되기 때문에 Counter 재사용이 불가능
- props로 전달한다면? → 컴포넌트가 깊게 중첩되면서 prop drilling 발생

```tsx
// 이런 식을 말하는 걸까...? 아래 코드로만 봤을 때는 책에서 말하는 우려하는 문제가 와닿지는 않는다.
const Counter = ({ store }) => {
  const [state, setState] = useStore(store)
  // ...
}

const Component = () => (
  <>
    <Counter store={store} />
    <Counter store={store2} />
  </>
)
```

- Counter 컴포넌트를 다른 스토어에서 재사용할 수 있도록 만들어보기 → 이번 챕터 목표

```tsx
// 공급자 컴포넌트만 변경되고 Counter 컴포넌트는 재사용
const Component = () => (
  <StoreProvider>
    <Counter />
    <Counter />
  </StoreProvider>
)

const Component2 = () => (
  <Store2Provider>
    <Counter />
    <Counter />
  </Store2Provider>
)

const Component3 = () => (
  <Store3Provider>
    <Counter />
    <Counter />
  </Store3Provider>
)
```

### 컨텍스트 사용이 필요한 시점

- 컨텍스트 공급자 → 기본 컨텍스트 값 또는 부모 공급자가 제공하는 값이 있는 경우 이 값을 **재정의(override)하는 메서드**
  - 적절한 기본값이 있다면 공급자를 사용할 이유가 없음
  - 전체 컴포넌트 트리의 하위 트리에 대해 다른 값을 제공할 필요가 있다면 공급자를 사용해야 함

> React 19 업데이트에서 컨텍스트 관련 내용

#### `use` API로 컨텍스트 읽기

- `use`로 컨텍스트 읽는 게 기존의 `useContext`와 다른 점? → 조건문이나 반복문 내부에서도 사용 가능
  - 기존 훅들이 컴포넌트 최상위에서만 사용할 수 있었던 걸 생각하면 신기한 훅임….
- 조건문 내부에서 사용하기

```tsx
function HorizontalRule({ show }) {
  if (show) {
    const theme = use(ThemeContext)
    return <hr className={theme} />
  }
  return null
}
```

- 얼리 리턴도 가능함

```tsx
import { use } from 'react'
import ThemeContext from './ThemeContext'

function Heading({ children }) {
  if (children == null) {
    return null
  }

  // 얼리 리턴되어 useContext 동작하지 않음
  const theme = use(ThemeContext)
  return <h1 style={{ color: theme.color }}>{children}</h1>
}
```

#### `<Context>`로 Provider 제공

- 기존 `<Context.Provider>` 같이 공급자 컴포넌트 대신 `<Context>`를 바로 사용하여 Provider 렌더링 가능

```tsx
const ThemeContext = createContext('')

function App({ children }) {
  return <ThemeContext value="dark">{children}</ThemeContext>
}
```

### 컨텍스트와 구독 패턴 사용하기

#### 컨텍스트 생성하기

- 기본 스토어의 상태 → `count`, `text`라는 두 가지 속성

```tsx
type State = { count: number; text?: string }

const StoreContext = createContext<Store<State>>(createStore<State>({ count: 0, text: 'hello' }))
```

#### 공급자 컴포넌트 구현하기

- 하위 트리에 서로 다른 스토어 제공 목적
- `useRef`를 사용한 이유? store 객체가 첫 번째 렌더링에서 한 번만 초기화되게 만드는 데 사용

```tsx
const StoreProvider = ({
  initialState,
  children,
}: {
  initialState: State
  children: ReactNode
}) => {
  const storeRef = useRef<Store<State>>()
  if (!storeRef.current) {
    storeRef.current = createStore(initialState)
  }
  return <StoreContext.Provider value={storeRef.current}>{children}</StoreContext.Provider>
}
```

#### `useSelector`로 스토어 객체 사용하기

- `StoreContext`로 `store` 객체를 가져오고, `selector`를 통해 원하는 데이터 반환
- 자체적으로 상태와 구독 관리를 최적화해주는 `useSyncExternalStore` 사용 → `useSubscription`, `useMemo` 제거

```tsx
const useSelector = <S extends unknown>(selector: (state: State) => S) => {
  const store = useContext(StoreContext)
  return useSyncExternalStore(store.subscribe, () => selector(store.getState()))
}
```

#### 컨텍스트+구독 패턴 사용하기

- 사용 전, 컨텍스트를 사용해서 상태를 갱신하는 useSetState훅을 만들어줘야 함

```tsx
const useSetState = () => {
  const store = useContext(StoreContext)
  return store.setState
}
```

```tsx
const selectCount = (state: State) => state.count // 컴포넌트 외부에서 선택자 함수 정의

const Component = () => {
  const count = useSelector(selectCount)
  const setState = useSetState()
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }))
  }
  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  )
}
```

> 이렇게 만든 컴포넌트는 **특정 스토어 객체에 연결돼 있지 않다** → store 주입이 없음
> 그러므로, 다양한 위치에서 컴포넌트를 가질 수 있다.

- 공급자 외부
- 첫 번째 공급자 내부
- 두 번째 공급자 내부
  >

#### 서로 다른 `StoreProvider` 컴포넌트 내에서 상태 값 확인하기

```tsx
const App = () => (
  <>
    <h1>Using default store</h1>
    {/* 아래 두 컴포넌트는 초기 count 값이 0으로 설정 */}
    <Component />
    <Component />
    <StoreProvider initialState={{ count: 10 }}>
      <h1>Using store provider</h1>
      {/* 아래 두 컴포넌트는 초기 count 값이 10으로 설정 */}
      <Component />
      <Component />
      <StoreProvider initialState={{ count: 20 }}>
        <h1>Using inner store provider</h1>
        {/* 아래 두 컴포넌트는 초기 count 값이 20으로 설정 */}
        <Component />
        <Component />
      </StoreProvider>
    </StoreProvider>
  </>
)
```

> **컨텍스트**로 인해 하위 트리에서 상태를 분리할 수 있고, **구독**으로 인해 리렌더링 문제를 피할 수 있다.
