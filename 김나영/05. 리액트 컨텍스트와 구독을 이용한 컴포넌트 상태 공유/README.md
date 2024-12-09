## 리액트 컨텍스트와 구독을 이용한 컴포넌트 상태 공유

- 모듈 상태 공유를 위해 구현한 createStore 함수를 통해 store을 생성할 수 있다.

```jsx
const store = createStore({ count: 0 });
```

**단점 : store을 사용하는 <Counter />컴포넌트는 재사용가능해야하지만, 모듈 상태는 리액트의 '외부'에서 정의되기 때문에 불가능함. -> 모듈상태의 한계**

#### 컨텍스트 사용이 필요한 시점

```jsx
const ThemeContext = createContext('light');

const Component = () => {
  const theme = useContext(ThemeContext);

  return <div>{theme}</div>;
};
```

- 공급자는 중첩될 수 있고, 가장 안쪽에 위치한 공급자의 값을 사용함

```jsx
<ThemeContext.Provider value='dark'>
  <ThemeContext.Provider value='dark'>
    <ThemeContext.Provider value='dark'>
      <Component />
    </ThemeContext.Provider>
  </ThemeContext.Provider>
</ThemeContext.Provider>
```

- 컴포넌트 트리에 공급자가 없는 경우에는 기본값을 사용한다.
- 공급자는 이 기본값 또는 부모 공급자의 값을 override하는 메서드이다.

#### 컨텍스트와 구독 패턴 사용하기

- 컨텍스트를 생성할 때 Context 값에 createStore (모듈 상태 만들기) 를 함께 사용해보자! count와 text라는 두가지 속성이 있다고 하자.

```jsx
const StoreContext = createContext(createStore({ count: 0, text: 'hi' }));
```

- 하위 트리에 서로 다른 스토어를 제공하기 위해 StoreProvider 구현

```jsx
const StoreProvider = (initialState, children) => {
  // NOTE: 첫 번째 렌더링에서 한번만 초기화되게 함
  const storeRef = useRef();

  if (!storeRef.current) {
    storeRef.current = createStore(initialState);
  }

  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
};
```

- useSelector로 상태를 활용할 때, 인수로 스토어 객체를 받는 것이 아닌 StoreContext에서 store객체를 가져와서 넣어줌

```jsx
const useSelector = (selector) => {
  const store = useContext(StoreContext);

  return useSubscription(
    useMemo(() => ({
      getCurrentValue: () => selector(store.getState()),
      subscribe: store.subscribe(),
    })),
  );
};
```

- 컨텍스트를 사용해 모듈 상태를 업데이트 하는 로직이 필요함

```jsx
const useSetState = () => {
  const store = useContext(StoreContext);

  return store.setState;
};
```

- 사용하기
  **컴포넌트 외부에서 SelectCount를 정의하지 않으면 함수를 useCallback 로 감싸야 하므로 추가작업이 발생한다?**

```jsx
const selectCount = (state) => state.count;

const Component = () => {
  const count = useSelector(selectorCount);
  const setState = useSetState();

  const inc = () => {
    setState((prev) => ({ ...prev, count: prev.count + 1 }));
  };

  return (
    <>
      <button onClick={inc}>
        <div>{count}</div>
      </button>
    </>
  );
};
```

- 하위트리에서 분리된 값을 제공하고
- 리렌더링을 피할 수 있음

### 추가 : Next.js에서 children 패턴을 사용했던 이유

- 작년 next.js 13버전을 처음 사용했을 때 recoil을 도입하려고 함
- recoilRoot를 사용하기 위해 layout.tsx에 'use client'를 사용해야하나.. 했으나
- RecoilProvider는 클라이언트 컴포넌트이고 children에 서버 컴포넌트가 들어오면 이 서버 컴포넌트들이 다 클라이언트 컴포넌트로 동작하는 것이 아닐까? 하는 의문점을 가졌음 ...
- Recoil의 컨텍스트가 사용되어 서버는 서버 컴포넌트를 유지하고, 리코일을 사용하는 컴포넌트만 client로 렌더링됨
- 작년엔 이해하지 못하고 사용했었는데 ....

### 추가 : React 19 에서 권장하는 Context의 방향

- 기존의 Context.Provider 대신 <Context> 자체를 Provider로 사용하는 방식 도입
- 이전방식

```jsx
import { createContext, useContext, useState } from 'react';

// Context 생성
const ThemeContext = createContext();

// 부모 컴포넌트
const ParentComponent = () => {
  const [theme, setTheme] = useState('dark');
  const changeTheme = (theme) => {
    setTheme(theme);
  };

  return (
    // Context.Provider를 사용하여 값 전달
    <ThemeContext.Provider value={{ theme, changeTheme }}>
      <ChildComponent />
    </ThemeContext.Provider>
  );
};

// 자식 컴포넌트
const ChildComponent = () => {
  const { theme } = useContext(ThemeContext);

  return <div className={theme}>Hello world!</div>;
};

export default ParentComponent;
```

- 변경된 방식

```jsx
import { createContext, useContext, useState } from 'react';

// Context 생성
const ThemeContext = createContext();

// 부모 컴포넌트
const ParentComponent = () => {
  const [theme, setTheme] = useState('dark');
  const changeTheme = (theme) => {
    setTheme(theme);
  };

  return (
    // Context 자체를 사용하여 값 전달
    <ThemeContext value={{ theme, changeTheme }}>
      <ChildComponent />
    </ThemeContext>
  );
};

// 자식 컴포넌트
const ChildComponent = () => {
  const { theme } = useContext(ThemeContext);
  return <div className={theme}>Hello world!</div>;
};

export default ParentComponent;
```

### 추가 : React 19 new hook : use ()

```jsx
import { use } from 'react';

const Button = () => {
  if (show) {
    const theme = use(ThemeContext);

    return <hr className={theme} />;
  }

  return false;
  // ...
};
```
