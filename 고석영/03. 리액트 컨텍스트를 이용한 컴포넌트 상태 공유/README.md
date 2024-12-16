### 컨텍스트(Context)

- 리액트 v16.3부터 제공
- 딱히 상태와 관련은 없지만 props를 대신해서 컴포넌트 간에 데이터 전달 가능
- 컨텍스트를 컴포넌트 상태와 결합하면 전역 상태를 제공 가능

> 즉, 컨텍스트는 전역 상태 관리 도구라기보다는, 컴포넌트 트리 깊숙한 곳까지 props를 전달하지 않고 필요한 데이터를 공유하기 위한 도구

### 컨텍스트 이해하기

- 컨텍스트 공급자가 새로운 컨텍스트 값을 갖게 되면 모든 컨텍스트 소비자는 새로운 값을 받고 리렌더링
- 컨텍스트 값이 변경되지 않았을 때 리렌더링 방지하는 방법
  - 내용 끌어올리기 → 상태 관리와 컨텍스트 분리를 통해 렌더링 범위 제한
  - `memo` → 메모이제이션된 컴포넌트로 만들어 다시 렌더링하지 않게 최적화
- 또한, 컨텍스트에 객체 값을 사용할 경우, 불필요한 리렌더링이 발생할 수 있음

### 전역 상태를 위한 컨텍스트 만들기

#### 작은 상태를 여러 조각으로 만들기

- 하나의 객체 값을 사용할 경우, 불필요한 리렌더링이 발생할 수 있음
- 따라서, 합쳐진 큰 객체를 사용하는 대신 각 조각에 대한 컨텍스트와 전역 상태를 사용하는 방법
- 결국 공급자 컴포넌트를 중첩해서 작성해야 하기 때문에 많으면 많을수록 중첩이 깊어짐
  - → 콜백 지옥처럼 에네르기파가 생길 수 있음…

#### useReducer로 하나의 상태를 만들고 여러 컨텍스트로 전파하기 → [공식문서에서 권장하는 방법](https://ko.react.dev/learn/scaling-up-with-reducer-and-context)

- 단일 상태를 만들고 여러 컨텍스트를 사용해 상태 조각을 배포하는 방법
- 여기서 중요한 점 → 상태(state)와 상태를 갱신하는 함수(dispatch)를 별도의 컨텍스트로!

```tsx
type Action = { type: 'INC1' } | { type: 'INC2' }

const Count1Context = createContext<number>(0)
const Count2Context = createContext<number>(0)
const DispatchContext = createContext<Dispatch<Action>>(() => {})

const Counter1 = () => {
  const count1 = useContext(Count1Context)
  const dispatch = useContext(DispatchContext)
  return (
    <div>
      Count1: {count1} <button onClick={() => dispatch({ type: 'INC1' })}>+1</button>
    </div>
  )
}

const Counter2 = () => {
  const count2 = useContext(Count2Context)
  const dispatch = useContext(DispatchContext)
  return (
    <div>
      Count2: {count2} <button onClick={() => dispatch({ type: 'INC2' })}>+1</button>
    </div>
  )
}

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
)

const Provider = ({ children }: { children: ReactNode }) => {
  const [state, dispatch] = useReducer(
    (prev: { count1: number; count2: number }, action: Action) => {
      if (action.type === 'INC1') {
        return { ...prev, count1: prev.count1 + 1 }
      }
      if (action.type === 'INC2') {
        return { ...prev, count2: prev.count2 + 1 }
      }
      throw new Error('no matching action')
    },
    {
      count1: 0,
      count2: 0,
    },
  )
  return (
    <DispatchContext.Provider value={dispatch}>
      <Count1Context.Provider value={state.count1}>
        <Count2Context.Provider value={state.count2}>{children}</Count2Context.Provider>
      </Count1Context.Provider>
    </DispatchContext.Provider>
  )
}

const App = () => (
  <Provider>
    <Parent />
  </Provider>
)
```

- count1이 변경되면 Counter1만 리렌더링 되고 Count2는 리렌더링 되지 않음 → 불필요한 리렌더링 문제 해결

> 핵심은 불필요한 리렌더링을 피하기 위해 여러 컨텍스트를 사용하는 것이다.
> 즉, 불필요한 리렌더링으로 인한 성능 측면에서의 비용 문제와 여러 컨텍스트 사용으로 인한 가독성 측면에서의 비용 문제를 비교해서 적절히 취사선택 해야 하는 것 같다.

### 컨텍스트 사용을 위한 모범 사례

#### 커스텀 훅이 있는 팩토리 패턴

- `createStateContext`: 다소 반복적일 수 있는 커스텀 훅+공급자 컴포넌트를 만드는 팩토리 패턴의 유틸리티 함수
  - 상태를 가져오는 커스텀 훅 + 공급자 컴포넌트를 튜플로 반환

```tsx
const createStateContext = <Value, State>(useValue: (init?: Value) => State) => {
  const StateContext = createContext<State | null>(null)
  const StateProvider = ({
    initialValue,
    children,
  }: {
    initialValue?: Value
    children?: ReactNode
  }) => <StateContext.Provider value={useValue(initialValue)}>{children}</StateContext.Provider>
  const useContextState = () => {
    const value = useContext(StateContext)
    if (value === null) throw new Error('Provider missing')
    return value
  }
  return [StateProvider, useContextState] as const
}
```

#### reduceRight를 이용한 공급자 중첩 방지

```tsx
const App = () => (
  <Count1Provider initialValue={10}>
    <Count2Provider initialValue={20}>
      <Count3Provider initialValue={30}>
        <Count4Provider initialValue={40}>
          <Count5Provider initialValue={50}>
            <Parent />
          </Count5Provider>
        </Count4Provider>
      </Count3Provider>
    </Count2Provider>
  </Count1Provider>
)
```

- reducerRight으로 공급자 컴포넌트의 중첩 리팩토링

```tsx
const App = () => {
  const providers = [
    [Count1Provider, { initialValue: 10 }],
    [Count2Provider, { initialValue: 20 }],
    [Count3Provider, { initialValue: 30 }],
    [Count4Provider, { initialValue: 40 }],
    [Count5Provider, { initialValue: 50 }],
  ] as const
  return providers.reduceRight(
    (children, [Component, props]) => createElement(Component, props, children),
    <Parent />,
  )
}
```

> [!NOTE]
>
> - `reduce()` → 왼쪽에서 오른쪽
> - `reduceRight` → 오른쪽에서 왼쪽
>
> 함수형 프로그래밍에서 `pipe`, `compose`의 차이랑도 비슷하다..!
>
> - `pipe` → 왼쪽에서 오른쪽, 읽기 순서대로 실행되므로 데이터 흐름 파악에 용이
> - `compose` → 오른쪽에서 왼쪽, 함수 합성 원리에 기반한 코드 작성
> - 조합의 방향에 따라 값이 달라질 수 있으므로 코드 가독성을 위해 일관된 사용이 필요할 것으로 보임
>
> ```tsx
> const add = x => x + 1
> const multiply = x => x * 2
>
> const pipeResult = pipe(add, multiply)(2) // (2 + 1) * 2 = 6
> const composeResult = compose(add, multiply)(2) // 2 * 2 + 1 = 5
> ```
