> [!NOTE]
>
> - React Tracked는 속성 감지를 기반으로 자동으로 렌더링 최적화를 수행하는 상태 사용 추적 라이브러리
> - 불필요한 리렌더링 제거 기능이 주목적
> - 주로 상태 관리를 위해 `useState`, `useReducer`를 사용하지만 Redux, Zustand 및 다른 상태 관리 라이브러리와 함께 사용 가능

### React Tracked 이해하기

- 상태 관리 기능 제공하지 않음
- 렌더링 최적화 기능 제공 → 상태 사용 추적

```tsx
const NameContext = createContext([{ firstName: 'react', lastName: 'hooks' }, () => {}])

const NameProvider = ({ childrend }) => (
  <NameContext.Provider value={useState({ firstName: 'react', lastName: 'hooks' })}>
    {children}
  </NameContext.Provider>
)

const useFirstName = () => {
  const [{ firstName }] = useContext(NameContext)
  return firstName
}
```

#### `useTracked`

- `useContext(NameContext)`를 대체하여 사용 가능
- 상태 사용 추적의 핵심 → 코드는 평소와 똑같아 보이지만 내부에서 상태 사용을 추적하고 렌더링 자동 최적화

```tsx
const useFirstName = () => {
  const [{ firstName }] = useTracked()
  return firstName
}
```

- 상태를 프락시로 감싸고 사용을 추적

```tsx
// https://github.com/dai-shi/react-tracked/blob/main/src/createTrackedSelector.ts
import { createProxy, isChanged } from 'proxy-compare'

const useTrackedSelector = () => {
  // ...
  return createProxy(state, affected, proxyCache)
}
```

- Valtio와 동일하게 상태 사용 추적 기능을 위해 [proxy-compare](https://www.npmjs.com/package/proxy-compare) 사용

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/fc48ee3d-1b55-4437-92f8-fdb7b54dff50/4bcdca57-9e9d-4b8d-83c9-93af3f450725/image.png)

### useState + React Tracked

```tsx
// (1) useState를 호출하는 커스텀 훅 정의
const useValue = () => useState({ count: 0, text: 'hello' })
// (2) 컨텍스트 정의
const StateContext = createContext<ReturnType<typeof useValue> | null>(null)
// (3) Provider 생성
const Provider = ({ children }: { children: ReactNode }) => (
  <StateContext.Provider value={useValue()}>{children}</StateContext.Provider>
)
// (4) 컨텍스트 사용을 위한 커스텀 훅 정의
const useStateContext = () => {
  const contextValue = useContext(StateContext)
  if (contextValue === null) {
    throw new Error('Please use Provider')
  }

  return contextValue
}
```

```tsx
// (5) 컴포넌트 정의
const Counter = () => {
  const [state, setState] = useStateContext()
  const inc = () => {
    setState(prev => ({
      ...prev,
      count: prev.count + 1,
    }))
  }

  return (
    <div>
      count: {state.count}
      <button onClick={inc}>+1</button>
    </div>
  )
}

const TextBox = () => {
  const [state, setState] = useStateContext()
  const setText = (text: string) => {
    setState(prev => ({
      ...prev,
      text: text,
    }))
  }

  return (
    <div>
      <input value={state.text} onChange={e => setText(e.target.value)} />
    </div>
  )
}

const App = () => {
  return (
    <Provider>
      <Counter />
      <Counter />
      <TextBox />
      <TextBox />
    </Provider>
  )
}
```

- 컨텍스트는 상태 객체를 전체적으로 처리하기 때문에 상태 객체가 변경되면 `useContext`가 리렌더링 감지
- 상태 객체에서 속성 하나만 변경되더라도 모든 `useContext` 훅은 리렌더링 감지 → 이 과정에서 불필요한 리렌더링 발생
- 컨텍스트에서 불필요한 리렌더링 동작이 예상되고 이를 방지하려면 더 작은 조각으로 분할해야 함

#### `useTracked`으로 대체 시 개선점

```tsx
const { Provider, useTracked } = createContainer(useValue)
```

```tsx
const [state, setState] = useTracked()
```

- 대체 시 변경해야 할 코드가 거의 없음
  - `createContext` → `createContainer`
  - `useStateContext` → `useTracked`
- useTracked에 의해 반환된 state 객체가 추적됨 → useTracked 훅이 어떤 상태 속성에 접근했는지 기억한다는 의미
- 즉, **접근된 속성이 변경된 경우에만 리렌더링**하는 것으로 리렌더링 최적화 가능

### useReducer + React Tracked

```tsx
const useValue = () => {
  type State = { count: number; text: string }
  type Action = { type: 'INC' } | { type: 'SET_TEXT'; text: string }

  const [state, dispatch] = useReducer(
    (state: State, action: Action) => {
      switch (action.type) {
        case 'INC':
          return { ...state, count: state.count + 1 }
        case 'SET_TEXT':
          return { ...state, text: action.text }
        default:
          throw new Error('unknow action type')
      }
    },
    { count: 0, text: 'hello' },
  )

  useEffect(() => {
    console.log('latest state', state)
  }, [state])

  return [state, setState] as const
}
```

```tsx
const { Provider, useTracked } = createContainer(useValue)

const Count = () => {
  const [state, dispatch] = useTracked()
  const inc = () => {
    dispatch({ type: 'INC' })
  }

  return (
    <div>
      count: {state.count}
      <button onClick={inc}>+1</button>
    </div>
  )
}

const TextBox = () => {
  const [state, dispatch] = useTracked()
  const setText = (text: string) => {
    dispatch({ type: 'SET_TEXT', text: text })
  }

  return (
    <div>
      <input value={state.text} onChange={e => setText(e.target.value)} />
    </div>
  )
}

const App = () => {
  ;<Provider>
    <Counter />
    <Counter />
    <TextBox />
    <TextBox />
  </Provider>
}
```

- React Tracked가 리렌더링을 최적화할 수 있는 이유
  - 상태 사용 추적뿐만 아니라 use-context-selector라는 내부 라이브러리 사용
  - selector 함수를 사용해 컨텍스트 값 구독 → 리액트 컨텍스트 제약을 우회

```tsx
// https://github.com/dai-shi/react-tracked/blob/main/src/createContainer.ts

import { createContext, useContextSelector, useContextUpdate } from 'use-context-selector'
import type { Context } from 'use-context-selector'

const useSelector = <Selected,>(selector: (state: State) => Selected) => {
  if (hasGlobalProcess && process.env.NODE_ENV !== 'production') {
    const selectorOrig = selector
    selector = (state: State) => {
      if (state === undefined) {
        throw new Error('Please use <Provider>')
      }
      return selectorOrig(state)
    }
  }
  const selected = useContextSelector(StateContext as Context<State>, selector)
  useDebugValue(selected)
  return selected
}
```

### React Redux + React Tracked

- React Tracked는 리액트 컨텍스트를 사용하지 않는 경우를 위해 `createTrackedSelector`라는 저수준 함수를 제공
- `useSelector` 훅을 받아 `useTrackedState`라는 훅을 반환

```tsx
const useTrackedState = createTrackedSelector(useSelector)
```

- createContainer는 이 함수에서 selector를 넣어주는 것만

#### Redux 스토어 생성하기

```tsx
import { createStore } from 'redux'
import { Provider, useDispatch, useSelector } from 'react-redux'
import { createTrackedSelector } from 'react-tracked'

type State = { count: number; text: string }
type Action = { type: 'INC' } | { type: 'SET_TEXT'; text: string }

const initialState: State = { count: 0, text: 'hello' }
const reducer = (state = initialState, action: Action) => {
  switch (action.type) {
    case 'INC':
      return { ...state, count: state.count + 1 }
    case 'SET_TEXT':
      return { ...state, text: action.text }
    default:
      return state
  }
}

const store = createStore(reducer)
```

#### `useTrackedState` 훅 정의하기

- `createTrackedSelector`로 react-redux에서 가져온 `useSelector` 훅을 이용해 생성
- 추적하려는 속성 파악을 위해 `State`로 훅의 타입 명시적으로 지정해야 함

```tsx
const useTrackedState = createTrackedSelector<State>(useSelector)
```

#### 컴포넌트 정의하기

```tsx
const Counter = () => {
  const dispatch = useDispatch()
  const { count } = useTrackedState()
  const inc = () => dispatch({ type: 'INC' })

  return (
    <div>
      count: {count} <button onClick={inc}>+1</button>
    </div>
  )
}

const TextBox = () => {
  const dispatch = useDispatch()
  const state = useTrackedState()
  const setText = (text: string) => dispatch({ type: 'SET_TEXT', text })

  return (
    <div>
      <input value={state.text} onChange={e => setText(e.target.value)} />
    </div>
  )
}

const App = () => (
  <Provider store={store}>
    <div>
      <Counter />
      <Counter />
      <TextBox />
      <TextBox />
    </div>
  </Provider>
)
```
