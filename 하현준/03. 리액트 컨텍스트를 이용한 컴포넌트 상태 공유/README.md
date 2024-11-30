리액트에서 컨텍스트(Context)기능을 제공하기 시작했고, 이를 사용하면 전역 상태를 사용할 수 있게 된다. 그러나 **리액트 컨텍스트는 전역 상태를 위해 설계된 것은 아니다.(중요!)**

특히 컨텍스트를 사용하는 소비자(Consumer)의 경우 리렌더링이 될 수 있는 존재이기 때문에 잘게 나누어서 사용해야 한다.

### 컨텍스트 이해하기

책에서 컨텍스트 사용법에 대해서 이야기 하는데 이건 [**공식 문서**](https://ko.react.dev/reference/react/useContext)를 살펴보는게 더 유용하니 넘어간다.

위에서 컨텍스트를 소비하는 컴포넌트는 리렌더링된다고 이야기 했다. 그 이유는 다음과 같다.

- 부모 컴포넌트 때문
- 컨텍스트 때문

컨텍스트가 전파되는 것을 알아보기 위한 예시)

```tsx
const ColorContext = createContext("black");

const ColorComponent = () => {
  const color = useContext(ColorContext);
  const renderCount = useRef(1);

  useEffect(() => {
    renderCount.current += 1;
  });

  return (
    <div style={{ color }}>
      Hello {color} (renders: {renderCount.current})
    </div>
  );
};

const MemoedColorComponent = memo(ColorComponent);

// Context를 사용하지 않은 컴포넌트
const DummyComponent = () => {
  const renderCount = useRef(1);
  useEffect(() => {
    renderCount.current += 1;
  });
  return <div>Dummy (renders: {renderCount.current})</div>;
};

// memo 처리를 한 컴포넌트
const MemoedDummyComponent = memo(DummyComponent);

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
  const [color, setColor] = useState("red");
  return (
    <ColorContext.Provider value={color}>
      <input value={color} onChange={(e) => setColor(e.target.value)} />
      <Parent />
    </ColorContext.Provider>
  );
};

export default App;
```

입력을 했을 때 어떻게 동작할까?

- 처음에 모든 컴포넌트를 렌더링
- input 값을 변경하면 당연히 App 컴포넌트 리렌더링
- color를 받은 동시에 Parent 리렌더링
- DummyComponent는 부모가 리렌더링 되어 리렌더링, MemoedDummyComponent는 리렌더링 되지 않음
- ColorComponent는 부모도 리렌더링 되었고, 컨텍스트도 바뀌어 당연히 리렌더링
- MemoedColorComponent는 컨텍스트가 바뀌어 리렌더링

## 컨텍스트에 객체를 사용했을 때의 한계점

https://github.com/wikibook/msmrh/blob/main/chapter03/05_pitfall-when-using-context-for-object/src/App.tsx

count1 또는 count2 만 변경했지만 모두 리렌더링 되버리는 문제가 있다.

이를 해결하기 위해서는 **컨텍스트를 작게 쪼개거나, useReducer로 하나의 상태를 만들어 여러 컨텍스트에 전파하는 것**이다.

결국에는 하나의 컨텍스트를 사용하지 않고 나눠야 한다는 것이다.

### 작게 쪼개기

```tsx
// https://github.com/wikibook/msmrh/blob/main/chapter03/06_creating-small-state-pieces/src/App.tsx

type CountContextType = [number, Dispatch<SetStateAction<number>>];

const Count1Context = createContext<CountContextType>([0, () => {}]);
const Count2Context = createContext<CountContextType>([0, () => {}]);

//...

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);
```

두개의 컨텍스트를 만들어서 제공하면 각각의 상태가 변할때만 리렌더링이 일어나게 된다.

### **useReducer로 하나의 상태를 만들어 여러 컨텍스트에 전파**

https://github.com/wikibook/msmrh/blob/main/chapter03/07_creating-one-state-with-userreducer-and-propagate-with-multiple-contexts/src/App.tsx

```tsx
type Action = { type: "INC1" } | { type: "INC2" };

const Count1Context = createContext<number>(0);
const Count2Context = createContext<number>(0);
const DispatchContext = createContext<Dispatch<Action>>(() => {});

// ...

const Provider = ({ children }: { children: ReactNode }) => {
  const [state, dispatch] = useReducer(
    (prev: { count1: number; count2: number }, action: Action) => {
      if (action.type === "INC1") {
        return { ...prev, count1: prev.count1 + 1 };
      }
      if (action.type === "INC2") {
        return { ...prev, count2: prev.count2 + 1 };
      }
      throw new Error("no matching action");
    },
    {
      count1: 0,
      count2: 0,
    }
  );
  return (
    <DispatchContext.Provider value={dispatch}>
      <Count1Context.Provider value={state.count1}>
        <Count2Context.Provider value={state.count2}>
          {children}
        </Count2Context.Provider>
      </Count1Context.Provider>
    </DispatchContext.Provider>
  );
};
```

상태와 실행 함수를 별도의 컨텍스트로 분리해 관리한다는 점이 키포인트이다.

이러한 방식을 나는 [모달 관리하는 방식을 작성해둔 블로그](https://nakta.dev/how-to-manage-modals-1)에서 참고해서 만들었던 기억이 있다. (내가 작성한 글: https://github.com/seulgi0711/site/issues/3#issuecomment-1797813977)

## 컨텍스트 다루는 패턴들

### 사용자 정의 훅과 공급자 컴포넌트 만들기

```tsx
// https://github.com/wikibook/msmrh/blob/main/chapter03/08_creating-custom-hook-and-provider-component/src/App.tsx

type CountContextType = [number, Dispatch<SetStateAction<number>>];

const Count1Context = createContext<CountContextType | null>(null);
export const Count1Provider = ({ children }: { children: ReactNode }) => (
  <Count1Context.Provider value={useState(0)}>
    {children}
  </Count1Context.Provider>
);
export const useCount1 = () => {
  const value = useContext(Count1Context);
  if (value === null) throw new Error("Provider missing");
  return value;
};
```

사용하는 쪽에서 useCount1() 로 컨텍스트의 존재를 숨겼기 때문에 사용하는 컴포넌트 쪽에서는 존재를 알지 못한다(이것이 추상화의 벽??🤩⇒ 쏙쏙.. 함수형)

### 사용자 정의 훅이 있는 팩토리 패턴

```tsx
// https://github.com/wikibook/msmrh/blob/main/chapter03/09_factory-pattern-with-custom-hook/src/App.tsx

// 일반화된 형태의 Provider/Context를 만드는 함수
const createStateContext = <Value, State>(
  useValue: (init?: Value) => State
) => {
  const StateContext = createContext<State | null>(null);

  const StateProvider = ({
    initialValue,
    children,
  }: {
    initialValue?: Value;
    children?: ReactNode;
  }) => (
    <StateContext.Provider value={useValue(initialValue)}>
      {children}
    </StateContext.Provider>
  );

  const useContextState = () => {
    const value = useContext(StateContext);
    if (value === null) throw new Error("Provider missing");
    return value;
  };

  return [StateProvider, useContextState] as const;
};

const useNumberState = (init?: number) => useState(init || 0);

const [Count1Provider, useCount1] = createStateContext(useNumberState);
```

위에서 소개하는 방식은 결국 반복 노가다 작업이 이뤄지기 때문에 이를 쉽게 할 수 있도록 팩토리(factory)를 하나 만들어서 계속해서 찍어낼 수 있게 하는 방식이다.

또한 타입스크립트와의 호환성도 굿굿 👍

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/253d1ac1-0c8d-4179-8d90-21ade38e0aea/245a1231-b45d-4187-9ac3-689db89931c4/image.png)

### reduceRight을 이용한 공급자 중첩 방지

즉 말그대로 공급자(provider)를 계속해서 감싸다보면 콜백 지옥과 같은 느낌을 받게 해주기 때문에 이를 `reduceRight`을 사용해 해결하는 것이다.

❓: 여기서 꼭 `reduceRight`이여야 하나라는 생각이 들어 사용법을 찾아보았다.

reduceRight으로 소개한 이유

- `reduceRight`을 쓰지 않으면 왼쪽부터 실행되기 때문에 배열을 거꾸로 작성해야하는 불편함이 있음
- `reduce`를 사용할 경우에도 해결은 되지만 휴먼 에러를 줄이기 위한 용도라고 생각이 된다.
  - 그런데 `reduce`사용법에 더 익숙한 사람은 reduce로 작성해도 무방할 것 같음

```tsx
const App = () => {
  // 방법1. reduceRight를 사용
  const providers = [
    [Count1Provider, { initialValue: 10 }],
    [Count2Provider, { initialValue: 20 }],
    [Count3Provider, { initialValue: 30 }],
    [Count4Provider, { initialValue: 40 }],
    [Count5Provider, { initialValue: 50 }],
  ] as const;

  return providers.reduceRight(
    (children, [Component, props]) => createElement(Component, props, children),
    <Parent />
  );

  // 방법2. reduce를 사용
  // 반대로 작성해서 사용해야 함
  const providers = [
    [Count5Provider, { initialValue: 50 }],
    [Count4Provider, { initialValue: 40 }],
    [Count3Provider, { initialValue: 30 }],
    [Count2Provider, { initialValue: 20 }],
    [Count1Provider, { initialValue: 10 }],
  ] as const;

  return providers.reduce(
    (children, [Component, props]) => createElement(Component, props, children),
    <Parent />
  );
};
```
