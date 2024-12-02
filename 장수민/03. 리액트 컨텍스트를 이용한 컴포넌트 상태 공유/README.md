## 3. 리액트 컨텍스트를 이용한 컴포넌트 상태 공유

리액트는 16.3 버전부터 컨텍스트(Context)라는 기능을 제공하기 시작했다. `useContext`라는 훅도 같이 제공하기 시작했는데, `useState`(or `useReducer`)를 같이 이용하면 전역 상태를 위한 사용자 정의 훅을 만들 수 있다.

> [!CAUTION]
> 컨텍스트는 사실 전역 상태를 위해 설계된 것은 아니다. 상태가 갱신될 때 모든 컨텍스트 소비자(consumer)가 리렌더링되므로 불필요한 렌더링이 발생할 수 있으므로, 전역 상태를 여러 조각으로 나누어 사용되는 것이 권장된다.

### 정적 값을 이용해 useContext 사용하기

```tsx
const ColorContext = createContext("black");

// 소비자 컴포넌트
const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};

const App = () => (
  <>
    {/* 공급자로 둘러싸여 있지 않아서 black이라는 정적 값 보여줌 */}
    <Component />
    <ColorContext.Provider value="red">
      <Component /> {/* red 표시 */}
    </ColorContext.Provider>
    <ColorContext.Provider value="green">
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
