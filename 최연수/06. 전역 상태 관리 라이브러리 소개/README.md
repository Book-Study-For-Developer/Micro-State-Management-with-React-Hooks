# 06. 전역 상태 관리 라이브러리 소개

> 전역 상태 관리 문제 해결, 데이터 중심 접근 방식 vs. 컴포넌트 중심 접근 방식, 리렌더링 최적화

<br/>

## 🔖 1. 전역 상태 설계 시 두 가지 문제점

### (1) 전역 상태를 읽는 방법

앞 장에서도 계속 언급하듯, 전역 상태를 사용하는 컴포넌트는 전역 상태의 모든 값이 필요하지 않은 경우가 있다. 전역 상태가 바뀌면 리렌더링이 발생하는데, 변경된 값이 컴포넌트와 관련 없는 경우에도 리렌더링이 발생한다. 이렇듯 비효율적인 경우에 전역 상태 라이브러리가 해결책을 제공한다.

뒤의 내용에서 설명하듯, 전역 상태 라이브러리들이 `selector`나 atom-like 구조를 통해 이 문제를 해결하려는 이유이기도 하다.

### (2) 전역 상태에 값을 넣거나 갱신하는 방법

전역 상태 중첩 객체를 직접 변경하는 것은 추적하기 어렵고, 불변성 유지를 깨트릴 수 있다. 개발자가 직접 전역 변수 값을 변경하기보다는 전용 함수를 제공하거나, 클로저를 활용해 변수에 직접 접근할 수 없도록 하는 방식은 상태 변경을 명확하고 안정적으로 관리하기도 한다.

### 두 문제를 해결하는 예시 코드

아래 예제의 전역 상태 `store`는 `user`라는 중첩 객체를 가지고 있고, `setUserName` 함수를 통해 `user.name` 값을 안전하게 변경한다. 이렇게 하면 전역 상태를 직접 변경하는 대신 전용 함수를 통해 변화가 일어나므로 관리와 추적이 용이해진다.

```javascript
// 전역 상태를 관리하는 store 구현
const createStore = (initialState) => {
  let state = initialState;
  const listeners = new Set();

  // 전역 상태를 읽는 함수
  const getState = () => state;

  // 전역 상태를 갱신하는 함수
  const setState = (newState) => { // 
    state = { ...state, ...newState };
    // 상태 변경 시 등록된 리스너(구독자)에게 알림
    listeners.forEach((listener) => listener());
  };

  const subscribe = (listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  };

  return { getState, setState, subscribe };
};

// 초기 전역 상태: user는 중첩 객체
const store = createStore({
  user: {
    name: 'Ella',
    age: 20
  },
  theme: 'dark'
});

// 전역 상태 변경을 위한 전용 함수 제공
function setUserName(newName) {
  // 전역 상태의 user 객체를 불변성을 유지하며 갱신
  const current = store.getState();
  store.setState({
    user: { 
      ...current.user,
      name: newName 
    }
  });
}

// 사용 예시: user.name을 직접 변경하지 않고 전용 함수로 변경
setUserName('Chloe');
console.log(store.getState().user.name); // "Chloe"
```

<br/>

## 🔖 2. 데이터 중심 접근 방식과 컴포넌트 중심 접근 방식 사용하기

> 나만의 언어로 paraphrasing하며 이해하기

### 데이터 중심 접근 방식

- 데이터 중심 접근 방식은 모듈 상태가 리액트 외부의 자바스크립트 메모리에서 관리된다.
- 이는 리액트 렌더링이 시작되기 전이나, 모든 컴포넌트가 사라진(언마운트) 뒤에도 그 상태가 유지될 수 있음을 의미한다.

### 컴포넌트 중심 접근 방식

- 전역 상태를 컴포넌트의 생명주기와 함께 관리한다.
- 전역 상태를 참조하는 컴포넌트들이 모두 사라지면(언마운트) 그 상태도 자동으로 없어지게 할 수 있다.
- 이 방식을 통해 같은 형태의 전역 상태를 여러 개 만들 수도 있는데, 각 전역 상태 인스턴스는 서로 다른 컴포넌트 트리에 소속될 수 있다.

### 팩토리 함수

- 데이터 중심 접근 방식을 사용하는 전역 상태 라이브러리는 "팩토리 함수"를 제공한다.
- 팩토리 함수 자체가 전역 상태를 바로 생성하는 것은 아니지만, 이 함수를 통해 만든 전역 상태 초기화 함수를 이용하면, 리액트가 해당 전역 상태의 생명주기를 책임지고 관리하도록 할 수 있다.

### 팩토리 함수? 팩토리 패턴?

> "팩토리 패턴(Factory Pattern)은 객체 생성 로직을 별도의 '팩토리'로 추상화하는 설계 패턴이다. 이때 '팩토리'라는 개념을 구현하는 한 가지 방법이 팩토리 함수(Factory Function)를 사용하는 것이다. 즉, 팩토리 패턴이라는 개념적 틀 안에서, 객체(또는 컴포넌트)를 생성하는 역할을 하는 함수를 팩토리 함수라고 부를 수 있다."

- [Dynamic Component Creation in React with the Factory Design Pattern](https://medium.com/@hossein.khoshnevis77/dynamic-component-creation-in-react-with-the-factory-design-pattern-d7fd2cf0101a)
- [The Factory Pattern in React @Zalando Engineering Blog](https://engineering.zalando.com/posts/2016/07/the-factory-pattern-in-react.html)

### 컴포넌트 중심 접근 예시 코드

컴포넌트 중심 접근 방식을 간략히 보여주는 예시 코드를 짜보았다. 특정 컴포넌트가 마운트될 때 전역 상태(store)를 초기화하고, 컴포넌트가 언마운트되면 그 전역 상태를 정리한다. 이로써 동일한 전역 상태 스키마를 가진 스토어를 여러 컴포넌트 트리에 각각 둘 수 있다.

```jsx
import React, { createContext, useContext, useEffect, useRef } from 'react';

// 전역 상태를 생성하는 팩토리 함수 (스키마는 같지만 인스턴스는 다르게 생성)
function createStoreFactory() {
  return function createStoreInstance(initialState) {
    let state = initialState;
    const listeners = new Set();

    return {
      getState: () => state,
      setState: (newState) => {
        state = { ...state, ...newState };
        listeners.forEach((listener) => listener());
      },
      subscribe: (listener) => {
        listeners.add(listener);
        return () => listeners.delete(listener);
      },
    };
  };
}

// 팩토리 함수 생성
const createStore = createStoreFactory();

// 컨텍스트 생성 (전역 상태 접근용)
const StoreContext = createContext(null);

// 이 컴포넌트가 마운트될 때 스토어 인스턴스를 생성하고, 언마운트될 때 정리
function MyComponentTreeProvider({ initialState, children }) {
  const storeRef = useRef(null);
  if (!storeRef.current) {
    storeRef.current = createStore(initialState);
  }

  useEffect(() => {
    // 마운트 시 storeRef.current가 유효하며
    // 언마운트 시 별도로 정리할 로직 (필요하다면) 여기서 가능
    return () => {
      // 필요하다면 storeRef.current 정리
      // 현재 예제에서는 특별한 정리 로직은 없다
    };
  }, []);

  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
}

// 전역 상태를 읽는 훅
function useGlobalState(selector) {
  const store = useContext(StoreContext);
  const [selectedValue, setSelectedValue] = React.useState(() => selector(store.getState()));

  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      const nextValue = selector(store.getState());
      setSelectedValue(nextValue);
    });
    return unsubscribe;
  }, [store, selector]);

  return selectedValue;
}

// 전역 상태를 갱신하는 훅
function useSetGlobalState() {
  const store = useContext(StoreContext);
  return store.setState;
}

// 예시 컴포넌트
function Counter() {
  const count = useGlobalState(state => state.count);
  const setState = useSetGlobalState();
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setState({ count: count + 1 })}>+1</button>
    </div>
  );
}

// 동일한 구조의 전역 상태를 다른 컴포넌트 트리에 각각 적용
function App() {
  return (
    <div>
      <h1>Component-based Global State</h1>
      <h2>Instance 1</h2>
      <MyComponentTreeProvider initialState={{ count: 0 }}> {/*0부터 1씩 증가*/}
        <Counter />
        <Counter />
      </MyComponentTreeProvider>

      <h2>Instance 2</h2>
      <MyComponentTreeProvider initialState={{ count: 10 }}> {/*10부터 1씩 증가*/}
        <Counter />
        <Counter />
      </MyComponentTreeProvider>
    </div>
  );
}

export default App;
```

<br/>

## 🔖 3. 리렌더링 최적화

> 리렌더링 최적화의 핵심은 컴포넌트에서 state의 어느 부분이 사용될지 지정하는 것인데, state의 일부분을 지정하는 세 가지 접근 방식이 있다. 나만의 언어로 paraphrasing하며 이해해보자.

### 선택자 함수 사용

- 리렌더링을 최적화하려면, 컴포넌트가 어떤 부분의 상태를 실제로 사용하는지 명확하게 하는 것이 중요하다.
- 이 중 하나의 접근 방식인 "선택자 함수"를 활용하면, 상태 전체가 아니라 **필요한 일부분만 추출**할 수 있다.
- 선택자 함수는 상태를 입력으로 받아 특정 값(혹은 파생된 값)을 반환한다.
- 이때, 같은 입력에 대해 항상 같은 결과를 내도록 함으로써, 상태 변경이 있어도 변하지 않는 부분은 재렌더링을 피할 수 있다.
- 이렇듯,  필요한 부분을 명시적으로 지정하는 과정을 '수동 최적화'라고 한다.

```jsx
import React, { useContext, useState } from 'react';

// 전역 상태 컨텍스트
const StateContext = React.createContext();

// 선택자 함수: 상태에서 count 값만 추출
function selectCount(state) {
  return state.count; // 여기
}

function useSelector(selector) {
  const state = useContext(StateContext);
  // selector 함수를 호출해 필요한 부분만 반환
  return selector(state);
}

function CounterDisplay() {
  // 현재 상태에서 count만 가져오기
  const count = useSelector(selectCount);
  return <div>Count: {count}</div>;
}

function App() {
  const [state, setState] = useState({ count: 0, text: 'hello' });

  return (
    <StateContext.Provider value={state}>
      <CounterDisplay />
      <button onClick={() => setState((prev) => ({ ...prev, count: prev.count + 1 }))}>
        Increment
      </button>
      <button onClick={() => setState((prev) => ({ ...prev, text: prev.text + '!' }))}>
        Change Text
      </button>
    </StateContext.Provider>
  );
}

export default App;
```

### 상태 사용 추적 (State Usage Traking)

- 상태 사용 추적이란, 컴포넌트가 전역 상태나 Store에서 어떤 프로퍼티(상태 값)에 접근하는지를 추적하는 기법이다. 이를 통해 컴포넌트가 실제로 참조한 상태 값이 변경될 때만 해당 컴포넌트를 리렌더링할 수 있게 된다.
- 즉, 컴포넌트가 렌더링될 때, 어떤 상태 필드(e.g. state.count, state.text)에 접근하는지 추적한다. 추적한 정보(어떤 필드를 읽었는지)를 바탕으로 컴포넌트가 해당 필드가 변경될 때만 리렌더링되도록 한다. 필드 변경이 필요 없는 경우에는 불필요한 리렌더링을 방지한다.
- 이 기법의 핵심은 상태 객체를 Proxy로 감싸고 '프로퍼티 접근'이 일어날 때마다 현재 렌더링 과정에서 해당 필드를 기록하는 것이다.

### State Usage Traking vs. Subscription

- 두 방식 모두 "필요한 상태가 변할 때만 컴포넌트를 리렌더링한다"는 점에서 비슷한 목표를 가지지만, 분명한 차이가 있다.
- Subscription은 개발자가 명시적으로 의존성을 선언하고 관리하는 '수동' 접근 방식이다.
- State Usage Traking은 코드 실행 과정에서 의존성을 '자동'으로 파악하는 방식이다.
- 두 방식 모두 불필요한 리렌더링을 방지하지만, State Usage Traking은 특히 더 자동화된 의존성 관리 덕분에 구조 변경에도 유연하게 대응할 수 있을 것으로 보인다.

### 아톰 사용 (using atom)

- 아톰은 atomic design의 atom에서도 유추할 수 있듯, 리렌더링을 발생시키는 데 사용되는 최소 상태 단위다.
- 전체 전역 상태를 구독해 리렌더링을 피하는 대신, 아톰을 사용하면 세분화해 구독하는 것이 가능하다.
- 아톰을 사용한 방식은 수동 최적화와 자동 최적화의 중간 정도로 볼 수 있다. 의존성 추적은 자동으로 되긴 한다.
