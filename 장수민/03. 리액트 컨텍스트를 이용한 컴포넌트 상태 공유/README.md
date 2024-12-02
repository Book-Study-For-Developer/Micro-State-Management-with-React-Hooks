## 3. 리액트 컨텍스트를 이용한 컴포넌트 상태 공유

리액트는 16.3 버전부터 컨텍스트(Context)라는 기능을 제공하기 시작했다. `useContext`라는 훅도 같이 제공하기 시작했는데, `useState`(or `useReducer`)를 같이 이용하면 전역 상태를 위한 사용자 정의 훅을 만들 수 있다.

> [!CAUTION]
> 컨텍스트는 사실 전역 상태를 위해 설계된 것은 아니다. 상태가 갱신될 때 모든 컨텍스트 소비자(consumer)가 리렌더링되므로 불필요한 렌더링이 발생할 수 있으므로, 전역 상태를 여러 조각으로 나누어 사용되는 것이 권장된다.

### 정적 값을 이용해 useContext 사용하기

```tsx
const ColorContext = createContext('black');

// 소비자 컴포넌트
const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};

const App = () => (
  <>
    {/* 공급자로 둘러싸여 있지 않아서 black이라는 정적 값 보여줌 */}
    <Component />
    <ColorContext.Provider value='red'>
      <Component /> {/* red 표시 */}
    </ColorContext.Provider>
    <ColorContext.Provider value='green'>
      <Component /> {/* green 표시 */}
    </ColorContext.Provider>
  </>
);
```

### useContext와 함께 useState 사용하기

```tsx
// count를 위한 컨텍스트 생성
const CountStateContext = createContext({
  count: 0,
  setCount: () => {},
});

const App = () => {
  const [count, setCount] = useState(0);
  return (
    <CountStateCountext.Provider value={{ count, setCount }}>
      <Parent />
    </CountStateContext.Provider>
  );
};

// 이제 props를 전달할 필요가 없다.
const Parent = () => (
  <>
    <Component1 />
    <Component2 />
  </>
);
```

> [!IMPORTANT]
> 이제 Component1, Component2는 가장 가까운 공급자로부터 컨텍스트 값을 가져오고, count와 setCount를 쓸 수 있게 된다.

### 컨텍스트 이해하기

컨텍스트 공급자를 사용할 경우 컨텍스트 값을 갱신할 수 있는데, 새로운 컨텍스트 값을 받으면 모든 컨텍스트 소비자 컴포넌트가 리렌더링된다.

컨텍스트 값이 변경되지 않았는데도 리렌더링이 일어나는 문제를 방지하려면 내용 끌어올리기 또는 `memo`를 쓰면 된다.

```tsx
const Parent = () => (
  <ul>
    <li>
      <DummyComponent />
    </li>
    <li>
      <MemoedDummyComponent />
    </li>
    <li>
      <ColorComponent />
    </li>
    <li>
      <MemoedColorComponent />
    </li>
  </ul>
);

const App = () => {
  const [color, setColor] = useState('red');
  return (
    <ColorContext.Provider value={color}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      <Parent />
    </ColorContext.Provider>
  );
};
```

위에 코드는 다음과 같이 동작한다.

1. 처음에 모든 컴포넌트가 렌더링된다.
2. 텍스트 입력 필드에서 값을 변경하면 App 컴포넌트가 리렌더링된다.
3. ColorContext.Provider는 새로운 값을 받고, 동시에 Parent 컴포넌트가 렌더링된다.
4. DummyComponent는 리렌더링되지만 MemoedDummyComponent는 리렌더링되지 않는다.
5. ColorComponent는 부모가 리렌더링되기도 했고, 컨텍스트가 변경되어 렌더링된다.
6. MemoedColorComponent는 컨텍스트가 변경됐기 때문에 리렌더링된다.

> [!IMPORTANT]
> 부모가 바뀌어서 컴포넌트가 리렌더링되거나, 컨텍스트가 변경되어 리렌더링되는 두 가지 경우를 잊지 말아야 한다. memo는 내부 컨텍스트 소비자가 리렌더링되는 것을 막지 못한다는 점도 중요하다.

### 컨텍스트에 객체를 사용할 때의 한계점

컨텍스트에서 관리하는 상태 값이 count1과 count2가 있다고 할 때, count1만 변경되어도 count2와 관련 있는 컴포넌트가 리렌더링되고 count2만 변경되어도 count1과 관련 있는 컴포넌트가 리렌더링 된다.

### 작은 상태 조각 만들기

```tsx
type CountContextType = [number, Dispatch<SetStateAction<number>>];

// 기본값으로 정적 값과 더미 함수를 지정
const Count1Context = createContext<CountContextType>([0, () => {}]);

// Count2Context도 상동

const Counter1 = () => {
  const [count1, setCount1] = useContext(Count1Context);
  return (
    <div>
      Count1: {count1}
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
    </div>
  );
};

// Counter2도 Counter1과 동일

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
);

const Count1Provider = ({ children }: { children: ReactNode }) => {
  const [count1, setCount1] = useState(0);
  return <Count1Context.Provider value={[count1, setCount1]}>{children}</Count1Context.Provider>;
};

// Count2Provider도 Count1Provider와 유사

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);
```

이제 각각 count1과 count2가 변경될 때만 리렌더링되어 이전의 문제가 발생하지 않는다. 따라서 각 상태에 대한 공급자를 만들 필요가 있다.

### useReducer로 하나의 상태를 만들고 여러 개의 컨텍스트로 전파하기

단일 상태를 만들고 여러 컨텍스트를 사용해 상태 조각을 배포하는 것이 두 번째 해결책이다.

```tsx
type Action = { type: 'INC1' } | { type: 'INC2' };

const Count1Context = createContext<number>(0);
const Count2Context = createContext<number>(0);
const DispatchContext = createContext<Dispatch<Aciton>>(() => {});

const Counter1 = () => {
  const count1 = useContext(Count1Context);
  const dispatch = useContext(DispatchContext);
  return (
    <div>
      Count1: {count1}
      <button onClick={() => dispatch({ type: 'INC1' })}>+1</button>
    </div>
  );
};

// Counter2는 Counter1과 유사

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
);

const Provider = ({ children }: { children: ReactNode }) => {
  const [state, dispatch] = useReducer(
    (prev: { count1: number; count2: number }, action: Action) => {
      if (action.type === 'INC1') return { ...prev, count1: prev.count1 + 1 };
      if (action.type === 'INC2') return { ...prev, count2: prev.count2 + 1 };
      throw new Error('no matching action');
    },
    {
      count1: 0,
      count2: 0,
    }
  );

  return (
    <DispatchContext.Provider value={dispatch}>
      <Count1Context.Provider value={state.count1}>
        <Count2Context.Provider value={state.count2}>{children}</Count2Context.Provider>
      </Count1Context.Provider>
    </DispatchContext.Provider>
  );
};

const App = () => (
  <Provider>
    <Parent />
  </Provider>
);
```

요점은 중첩된 공급자가 각 상태 조각과 '하나의 실행 함수'를 제공한다는 것이다.

### 모범 사례 - 사용자 정의 훅과 공급자 컴포넌트 만들기

```tsx
type CountContextType = [number, Dispatch<SetStateAction<number>>];

const Count1Context = createContext<CountContextType | null>(null);

export const CountProvider = ({ children }: { chidren: ReactNode }) => (
  <Count1Context.Provider value={useState(0)}>{children}</Count1Context.Provider>
);

// 사용자 정의 훅
export const useCount1 = () => {
  const value = useContext(Count1Context);
  if (value === null) throw new Error('Provider missing');
  return value;
};

const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Count1: {count1}
      <button onClick={() => setCount1((c) => c + 1)}>+1</button>
    </div>
  );
};
```

### 모범 사례 - 사용자 정의 훅이 있는 팩토리 패턴

사용자 정의 훅과 공급자 컴포넌트를 만드는 것은 다소 반복적인 작업이므로, 이런 작업을 수행하는 함수를 만들 수 있다.

```tsx
const createStateContext = (useValue: (init) => State) => {
  const StateContext = createContext(null);
  const StateProvider = ({ initialValue, children }) => (
    <StateContext.Provider value={useValue(initialValue)}>{children}</StateContext.Provider>
  );
  const useContextState = () => {
    const value = useContext(StateContext);
    if (value === null) throw new Error('Provider missing');
    return value;
  };
  return [StateProvider, useContextState] as const;
};

const [Count1Provider, useCount1] = createStateContext(useNumberState);

// 사용자 정의 훅인 useCount1을 사용하는 방법은 같다.
```

> [!TIP]
> 팩토리 패턴이 타입스크립트에서 잘 작동한다는 것이 주목할 만하다. 타입스크립트는 타입에 대한 추가적인 검사를 제공하며 개발자들은 타입 검사를 통해 더 나은 경험을 얻는다.

### 모범 사례 - reduceRight를 이용한 공급자 중첩 방지

5단계의 공급자 중첩으로 인한 코딩 스타일을 완화하기 위해 reduceRight를 사용할 수 있다.

```tsx
const App = () => {
  const providers = [
    [Count1Provider, { initialValue: 10 }],
    [Count2Provider, { initialValue: 20 }],
    [Count3Provider, { initialValue: 30 }],
    [Count4Provider, { initialValue: 40 }],
    [Count5Provider, { initialValue: 50 }],
  ] as const;
  return providers.reduceRight((children, [Component, props]) => createElement(Component, props, children), <Parent />);
};
```
