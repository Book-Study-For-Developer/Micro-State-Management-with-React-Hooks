## 03. 리액트 컨텐스트를 이용한 컴포넌트 상태 공유

> 컨텍스트는 전역 상태를 위해 설계된 것은 아니다. 상태가 갱신될 때 모든 컨텍스트 소비자 (consumer)가 리렌더링되므로 불필요한 리렌더링이 발생할 수 있다. 따라서 여러조각으로 나누어 사용하는것이 권장된다.

### 정적 값을 이용해 useContext 사용하기 

리액트 컨텍스트는 다양한 값을 제공하는 여러 개의 공급자(provider)가 있다. 공급자는 중첩될 수 있고 , 소비자는 컴포넌트(useContext가 있는 컴포넌트를 의미)는 컴포넌트 트리 중에서 가장 가까운 공급자를 선택해 컨텍스트 값을 가져온다. 

```js
import { createContext, useContext } from "react";

const ColorContext = createContext('black');

const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};

const App = () => (
  <>
    // 기본값 black 사용!
    <Component />
    <ColorContext.Provider value="red">
      // red값을 사용!
      <Component /> 
    </ColorContext.Provider>
    <ColorContext.Provider value="green">
      // green값을 사용!
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value="blue">
      // blue값을 사용!
      <Component />
      <ColorContext.Provider value="skyblue">
        // skyblue값을 사용!
        <Component />
      </ColorContext.Provider>
    </ColorContext.Provider>
  </>
);

export default App;
```

여러 공급자와 소비자 컴포넌트를 재사용하는 것은 리액트 컨텍스트의 중요한 기능이다.


### 컨텍스트 이해하기

> 컨텍스트 공급자가 새로운 컨텍스트 값을 갖게 되면 모든 컨텍스트 소비자는 새로운 값을 받고 리렌더링된다. 이는 공급자의 값이 모든 소비자에게 전파된다는 것을 의미한다. 

### 전역 상태를 위한 컨텍스트 만들기 

- 작은 상태 조각 만들기
- useReducer로 하나의 상태를 만들고 여러 컨텍스트로 전파하기

#### 작은 상태 조각 만들기 

```js
import {
  Dispatch,
  SetStateAction,
  ReactNode,
  createContext,
  useContext,
  useState,
} from "react";

type CountContextType = [number, Dispatch<SetStateAction<number>>];

const Count1Context = createContext<CountContextType>([0, () => {}]);
const Count2Context = createContext<CountContextType>([0, () => {}]);

const Counter1 = () => {
  const [count1, setCount1] = useContext(Count1Context);
  return (
    <div>
      Count1: {count1}{" "}
      <button onClick={() => setCount1((c) => c + 1)}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  const [count2, setCount2] = useContext(Count2Context);
  return (
    <div>
      Count2: {count2}{" "}
      <button onClick={() => setCount2((c) => c + 1)}>+1</button>
    </div>
  );
};

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
  return (
    <Count1Context.Provider value={[count1, setCount1]}>
      {children}
    </Count1Context.Provider>
  );
};

const Count2Provider = ({ children }: { children: ReactNode }) => {
  const [count2, setCount2] = useState(0);
  return (
    <Count2Context.Provider value={[count2, setCount2]}>
      {children}
    </Count2Context.Provider>
  );
};

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);

export default App;
```

위 챕터에서 본 예제와 달리 count를 사용하는 상태마다 context를 만들었고 해당 값을 변경시켰을때 불필요한 리렌더링을 발생시키지 않았다. 즉, 상태에 따른 각각의 공급자를 만들어서 사용했다. 

#### useReducer로 하나의 상태를 만들고 여러 개의 컨텍스트로 전파하기

```js
import {
  Dispatch,
  ReactNode,
  createContext,
  useContext,
  useReducer,
} from "react";

type Action = { type: "INC1" } | { type: "INC2" };

const Count1Context = createContext<number>(0);
const Count2Context = createContext<number>(0);
const DispatchContext = createContext<Dispatch<Action>>(() => {});

const Counter1 = () => {
  const count1 = useContext(Count1Context);
  const dispatch = useContext(DispatchContext);
  return ( 
    <div>
      Count1: {count1}{" "}
      <button onClick={() => dispatch({ type: "INC1" })}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  const count2 = useContext(Count2Context);
  const dispatch = useContext(DispatchContext);
  return (
    <div>
      Count2: {count2}{" "}
      <button onClick={() => dispatch({ type: "INC2" })}>+1</button>
    </div>
  );
};

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

const App = () => (
  <Provider>
    <Parent />
  </Provider>
);

export default App;
```

앞선 패턴들은 자주봤고 생소하지는 않았는데 dispatch context를 만들어서 reducer로 만든 부분이 굉장히 인상적이다. 

### 컨텍스트 사용을 위한 모범 사례

- 사용자 정의 훅과 공급자 컴포넌트 만들기
- 사용자 정의 훅이 있는 팩토리 패턴
- reduceRight를 이용한 공급자 중첩 방지

#### 사용자 정의 훅과 공급자 컴포넌트 만들기

```jsx
import {
  Dispatch,
  SetStateAction,
  ReactNode,
  createContext,
  useContext,
  useState,
} from "react";

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

const Count2Context = createContext<CountContextType | null>(null);
export const Count2Provider = ({ children }: { children: ReactNode }) => (
  <Count2Context.Provider value={useState(0)}>
    {children}
  </Count2Context.Provider>
);
export const useCount2 = () => {
  const value = useContext(Count2Context);
  if (value === null) throw new Error("Provider missing");
  return value;
};

const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Count1: {count1}{" "}
      <button onClick={() => setCount1((c) => c + 1)}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  const [count2, setCount2] = useCount2();
  return (
    <div>
      Count2: {count2}{" "}
      <button onClick={() => setCount2((c) => c + 1)}>+1</button>
    </div>
  );
};

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
);

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);

export default App;
```

useCount라는 hook에서 useContext를 사용하여 소비자 로직을 숨겼다.

#### 사용자 정의 훅이 있는 팩토리 패턴

```js
import { ReactNode, createContext, useContext, useState } from "react";

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
const [Count2Provider, useCount2] = createStateContext(useNumberState);

const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Count1: {count1}{" "}
      <button onClick={() => setCount1((c) => c + 1)}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  const [count2, setCount2] = useCount2();
  return (
    <div>
      Count2: {count2}{" "}
      <button onClick={() => setCount2((c) => c + 1)}>+1</button>
    </div>
  );
};

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
);

const App = () => (
  <Count1Provider>
    <Count2Provider>
      <Parent />
    </Count2Provider>
  </Count1Provider>
);

export default App;
```

createStateContext를 활용해서 반복적인 코드를 피하고 같은 기능을 제공하는 로직이 꽤나 인상 깊었다. 팩토리 패턴에 대해서 제대로 이해한건지 모르겠지만 아래 글을 봤을때 우리가 사용하는 대부분의 hook들은 팩토리 패턴인건가?

🔗 [Factory Method Pattern](https://refactoring.guru/ko/design-patterns/factory-method)

#### reduceRight를 이용한 공급자 중첩 방지

```js
import {
  ReactNode,
  createContext,
  createElement,
  useContext,
  useState,
} from "react";

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
const [Count2Provider, useCount2] = createStateContext(useNumberState);
const [Count3Provider, useCount3] = createStateContext(useNumberState);
const [Count4Provider, useCount4] = createStateContext(useNumberState);
const [Count5Provider, useCount5] = createStateContext(useNumberState);

const Counter1 = () => {
  const [count1, setCount1] = useCount1();
  return (
    <div>
      Count1: {count1}{" "}
      <button onClick={() => setCount1((c) => c + 1)}>+1</button>
    </div>
  );
};

const Counter2 = () => {
  const [count2, setCount2] = useCount2();
  return (
    <div>
      Count2: {count2}{" "}
      <button onClick={() => setCount2((c) => c + 1)}>+1</button>
    </div>
  );
};

const Parent = () => (
  <div>
    <Counter1 />
    <Counter1 />
    <Counter2 />
    <Counter2 />
  </div>
);

const App = () => {
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
};

/*
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
);
*/

export default App;
```

나는 사실 context API를 잘 활용하지 않아서 provider 지옥에 빠진적이 없었지만 종종 provider 지옥에 빠진 짤들을 봤었다. 위와 같은 방식으로 조금은 해결할 수 있을 것 같다는 생각이 들었다.


