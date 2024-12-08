# 3. 리액트 컨텍스트를 이용한 컴포넌트 상태 공유

## useContext를 활용하여 props 제거하기

```jsx
// props drilling
const App = () => {
  const [count, setCount] = useState(0);

  return <Parent count={count} setCount={setCount} />;
};

const Parent = ({ count, setCount }) => {
  <>
    <Component1 count={count} setCount={setCount} />
    <Component2 count={count} setCount={setCount} />
  </>;
};

// using Context
const ColorContext = createContext("black");

const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};

const App = () => (
  <>
    <Component />
    <ColorContext.Provider value="red">
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value="green">
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value="blue">
      <Component />
      <ColorContext.Provider value="skyblue">
        <Component />
      </ColorContext.Provider>
    </ColorContext.Provider>
  </>
);
```

→ 여러 Provider에서 가장 가까운 공급자를 선택하여 가져온다.

- React 내에서 어떻게 구현 되어 있을까
  `ReactFiberNewContext` 내부에서 `readContext` , `reactContextForConsumer` 를 통해 구현 되어 있음은 발견했지만 내부 구현에 대해서 완벽하게 이해하지는 못했다…

  ```jsx
  export function readContext<T>(context: ReactContext<T>): T {
    if (__DEV__) {
      // This warning would fire if you read context inside a Hook like useMemo.
      // Unlike the class check below, it's not enforced in production for perf.
      if (isDisallowedContextReadInDEV) {
        console.error(
          "Context can only be read while React is rendering. " +
            "In classes, you can read it in the render method or getDerivedStateFromProps. " +
            "In function components, you can read it directly in the function body, but not " +
            "inside Hooks like useReducer() or useMemo()."
        );
      }
    }
    return readContextForConsumer(currentlyRenderingFiber, context);
  }
  ```

  ```jsx
  function readContextForConsumer<C>(
    consumer: Fiber | null,
    context: ReactContext<C>
  ): C {
    const value = isPrimaryRenderer
      ? context._currentValue
      : context._currentValue2;

    const contextItem = {
      context: ((context: any): ReactContext<mixed>),
      memoizedValue: value,
      next: null,
    };

    if (lastContextDependency === null) {
      if (consumer === null) {
        throw new Error(
          "Context can only be read while React is rendering. " +
            "In classes, you can read it in the render method or getDerivedStateFromProps. " +
            "In function components, you can read it directly in the function body, but not " +
            "inside Hooks like useReducer() or useMemo()."
        );
      }

      // This is the first dependency for this component. Create a new list.
      lastContextDependency = contextItem;
      consumer.dependencies = __DEV__
        ? {
            lanes: NoLanes,
            firstContext: contextItem,
            _debugThenableState: null,
          }
        : {
            lanes: NoLanes,
            firstContext: contextItem,
          };
      if (enableLazyContextPropagation) {
        consumer.flags |= NeedsPropagation;
      }
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
    }
    return value;
  }
  ```

useContext와 useState 같이 사용하기

```jsx
const CountStateContext = createContext({
  count: 0,
  setCount: (_: SetStateAction<number>) => {},
});

const App = () => {
  const [count, setCount] = useState(0);
  return (
    <CountStateContext.Provider value={{ count, setCount }}>
      <Parent />
    </CountStateContext.Provider>
  );
};

const Parent = () => (
  <>
    <Component />
  </>
);

const Component = () => {
  const { count, setCount } = useContext(CountStateContext);
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
    </div>
  );
};
```

<br/>

## 컨텍스트 이해하기

###컨텍스트 전파의 작동 방식

Provider를 통해서 Context 값을 갱신할 수 있다. 새로운 Context 값을 받으면 Context의 소비자 컴포넌트는 리랜더링 된다.

컨텍스트가 변경 되지 않았음에도 리랜더링이 발생하면 ‘내용 끌어올리기 패턴’ 또는 `memo` 를 활용하여 리랜더링이 발생하는 것을 방지할 수 있다.

<br />

### 컨텍스트에 객체를 사용할 때의 한계점

객체 값을 사용할 때, 일부 상태 값만 활용하더라도 컴포넌트에서 사용하지 않는 다른 상태에 의해 리랜더링 발생할 수 있다.

→ **React는 기본적으로 객체를 비교할 때 얕은 비교(shallow comparison)를 수행한다**. 즉, **객체의 참조(reference)만 비교**하므로, 객체 내부의 `count1`이나 `count2` 중 하나만 변경되어도 **새로운 객체가 생성**되기 때문에 **다른 값이 변경되지 않았더라도 리렌더링이 발생한다.**

<br/>

## 전역 상태를 위한 컨텍스트 만들기

- 작은 상태로 나누어 전파하기

  ```tsx
  type CountContextType = [
  	number,
  	Dispatch<SetStateAction<number>>
  ];

  const Count1Context = createContext<CountContextType>([
  	0,
  	() => {}
  ])

  const Count2Context = createContext<CountContextType>([
  	0,
  	() => {}
  ])

  const Counter1 = () => {
  	const [ count1, setCount1 ] = useContext(Count1Context);
  	...
  }

  const Counter2 = () => {
  	const [ count2, setCount2 ] = useContext(Count2Context);
  	...
  }

  const Counter1  Provider = ({
  	children
  }) : {
  	children: ReactNode
  } => {
  	const [count1, setCount1 ] = useState(0);

  	return (
  		<Count1Context.Provider value={[ count1, setCount1 ]}>
  			{ children }
  		</Count1Context.Provider>
  	)
  }

  const Counter2Provider = ({
  	children
  }) : {
  	children: ReactNode
  } => {
  	const [count2, setCount2 ] = useState(0);

  	return (
  		<Count2Context.Provider value={[ count2, setCount2 ]}>
  			{ children }
  		</Count2Context.Provider>
  	)
  }
  ```

  → 공급자 컴포넌트가 많을수록 중첩이 많아진다.  
  → 각 상태에 마다 Provider를 만들어야한다.  
  → 상태가 많이 변하지 않거나 컨텍스트에 의존하는 컴포넌트가 없어서 컨텍스트 동작 방식에 문제가 없으면 해당 패턴을 사용하지 않아도 된다.

- useReducer로 하나의 상태를 관리하고, 여러 컨텍스트로 전파하기

  ```tsx
  type Action = { type: "INC1" } | { type: "INC2" };

  const Count1Context = createContext<number>(0);
  const Count2Context = createContext<number>(0);
  const DispatchContext = createContext<Dispatch<Action>>(() => {});

  const Counter1 = () => {
    const count1 = useContext(Count1Context);
    const dispatch = useContext(DispatchContext);

    return (
      <div>
        Count: {count1}
        <button onClick={() => dispatch({ type: "INC1" })}>+1</button>
      </div>
    );
  };

  const Provider = ({ children }: { children: ReactNode }) => {
    const [state, dispatch] = useReducer(
      (prev: { count1: number; count2: number }, action: Action) => {
        if (action.type === "INC1") {
          return { ...prev, count1: prev.count1 + 1 };
        }
        if (action.type === "INC2") {
          return { ...prev, count2: prev.count2 + 1 };
        }
        throw new Error();
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

  → 리듀서로 인해 코드 길이가 길어진다.
  → 하나의 액션으롤 여러 조각을 갱신할 수 있다.

<br/>

## 컨텍스트를 사용을 위한 모범 사례

- 사용자 정의 훅과 공급자 컴포넌트 활용

  ```tsx
  type CountContextType = [
  	number,
  	Dispatch<SetDispatchAction<number>>,
  ];

  const Count1Context = createContext<
  	CountContextType | null
  >(null);

  const Count1Provider = ({ children } : { children : ReactNode }) => {
  	return (
  		<Count1Context.Provider value={useState(0)}>
  			{ children }
  		</Count1Context.Provider>
  	)
  }

  const useCount1 = () => {
  	const value = useContext(Count1Context);

  	if ( value === null ) throw new Error("Provider missing");

  	return value;
  };

  const Count1 = () => {
  	const [ count1, setCount1 ] = useCount1();

  	return ...
  }
  ```

  - 사용자 정의 훅이 있는 팩토리 패턴
    사용자 정의 훅과 Provider를 **반복해서 만드는 작업을 간소화하기 위해** 이런 작업을 수행하는 함수를 만들어 사용할 수 있다.

    ```tsx
    const createStateContext = (
    	useValue: (init) => State, // 초기값을 통해 상태를 반환
    ) => {
    	// Context 생성
    	const StateContext = createContext(null);

    	// 공급자 컴포넌트
    	const StateProvider = ({
    		initialValue,
    		children
    	}) => (
    		// useState를 사용하여 반환된 state와 setState를 튜플을 그대로 전달
    		<StateContext.Provider value={useState(initialValue}>
    			{children}
    		</StateContext.Provider>
    	);

    	// 사용자 정의 훅
    	const useContextState = () => {
    		const value = useContext(StateContext);

    		if ( value === null ) throw new Error('Provider missing');

    		return value;
    	};

    	return [StateProvider, useContextState] as const;
    };

    // 사용할 때
    // useState
    const useNumberState = (init) => useState(init || 0 );
    // useReducer
    const useMyState = () => useReducer({}, (prev, action) => {
    	if (action.type === 'SET_FOO') {
    		return { ...prev, foo: action.foo };
    	}
    	// ...
    });

    const [Count1Provider, useCount1] = createStateContext(useNumberState);
    const [Count2Provider, useCount2] = createStateContext(useNumberState);
    ```

    → 팩토리 패턴은 TypeScript 환경에서도 잘 작동한다.

    ```tsx
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
    ```

<br/>

## reduceRight를 활용한 중첩 방지

생성자 컴포넌트가 많아지면 그만큼 컴포넌트 트리에 중첩이 많이 발생하게 된다.

reduceRight를 통해 중첩을 적용하지 않은 코딩 스타일로 리팩토링 가능하다.

```tsx
const App = () => {
	const provider = [
		[ Counter1Provider, { initalValue : 10 } ],
		[ Counter2Provider, { initalValue : 20 } ],
		[ Counter3Provider, { initalValue : 30 } ],
		[ Counter4Provider, { initalValue : 40 } ],
		...
	] as const;

	// reduceRight를 통한 중첩구조 생성
	return providers.reduceRight(
		(children, [ Component, props ]) =>
			createElement(Component, props, children),
		<Parent />,
	);
}
```

<details>
  <summary>이해를 돕는 중첩 구조 생성 과정</summary>
  
  ### **`reduceRight`의 동작 과정**
  #### 1. **초기 상태**:
  ```tsx
  children = <Parent />;
  ```
  #### 2. **첫 번째 반복** (`Counter4Provider`):
  ```tsx
  createElement(Counter4Provider, { initalValue: 40 }, <Parent />)

---

  <Counter4Provider initalValue={40}>
    <Parent />
  </Counter4Provider>
  ```
  #### 3. **두 번째 반복** (`Counter3Provider`):
  ```tsx
  createElement(Counter3Provider, { initalValue: 30 }, <Counter4Provider ...>)

---

  <Counter3Provider initalValue={30}>
    <Counter4Provider initalValue={40}>
      <Parent />
    </Counter4Provider>
  </Counter3Provider>
  ```
  #### 4. **세 번째 반복** (`Counter2Provider`):
  ```tsx
  createElement(Counter2Provider, { initalValue: 20 }, <Counter3Provider ...>)

---

  <Counter2Provider initalValue={20}>
    <Counter3Provider initalValue={30}>
      <Counter4Provider initalValue={40}>
        <Parent />
      </Counter4Provider>
    </Counter3Provider>
  </Counter2Provider>
  ```
  #### 5. **마지막 반복** (`Counter1Provider`):
  ```tsx
  createElement(Counter1Provider, { initalValue: 10 }, <Counter2Provider ...>)

---

  <Counter1Provider initalValue={10}>
    <Counter2Provider initalValue={20}>
      <Counter3Provider initalValue={30}>
        <Counter4Provider initalValue={40}>
          <Parent />
        </Counter4Provider>
      </Counter3Provider>
    </Counter2Provider>
  </Counter1Provider>
  ```
</details>

<br/>

### 더 알아보기 : `as const`를 꼭 써야하나?

```tsx
const App = () => {
  const provider = [
    [ Counter1Provider, { initalValue : 10 } ],
    [ Counter2Provider, { initalValue : 20 } ],
    [ Counter3Provider, { initalValue : 30 } ],
    [ Counter4Provider, { initalValue : 40 } ],
    ...
  ] as const;

  // reduceRight를 통한 중첩구조 생성
  return providers.reduceRight(
    (children, [ Component, props ]) =>
      createElement(Component, props, children),
    <Parent />,
  );
}
```

→ as const를 사용함으로서 providers 변수에 불변성을 부여

1.  리터럴 타입으로 변환

    ```tsx
    // as const 미사용
    [typeof CounterProvider, { initialValut: number }];

    // as const 사용
    readonly[(typeof Counter1Provider, { initialValue: 10 })];
    ```

2.  임의 수정 방지

        → as const를 통해 배열을 readonly로 만들어 실수로 수정되는 것을 방지할 수 있다.

        → 의도하지 않은 변경을 방지해 순수 함수와 불변성을 지키는데 도움이 된다.

    provider 배열을 통해 중첩 구조를 만들고 있기 때문에, 배열이나 객체가 임의로 수정이 가능하면 **불필요한 재렌더링**이나 **의도치 않은 상태 변화**로 인한 오류가 발생할 수 있다. 그래서 이 상황에서는 as const를 통해 불변성을 보장하려고 한 것 같다.

<br/>

### 더 알아보기 : reduceRight

→ reduce()와 같은 기능을 하지만 배열의 끝(오른쪽)에서 부터 시작한다.

```jsx
var arr = [10, 100, 1000];
var subtract = arr.reduce((total, value) => total - value);
var subtract_right = arr.reduceRight((total, value) => total - value);

console.log(subtract); // -1090
console.log(subtract_right); // 890
```

실행동작처럼 오른쪽에서부터 연산을 수행하려면 `.reverse()` 후에 `.reduce()` 를 적용하거나 했을 것 같은데 알아두면 쓸 일은 있지 않을까…?

**Reference**: https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Array/reduceRight
