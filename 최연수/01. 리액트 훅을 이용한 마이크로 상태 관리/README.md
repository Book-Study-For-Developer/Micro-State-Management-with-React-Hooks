# 01. 리액트 훅을 이용한 마이크로 상태 관리

> 리액트 훅으로 주목받은 마이크로 상태 관리의 개념, useState와 useReducer 훅의 기술적 측면

<br/>

## 🔖 1. 마이크로 상태 관리란 무엇인가

### 상태 관리의 유래(?)

개발을 역사처럼 공부하는 것을 좋아하는데, 1장 초반에 상태 관리를 어떻게 해왔는지 간략하게 설명해주어서 좋았다.

요약하자면 React는 컴포넌트 단위로 개발하는 라이브러리이며, React의 hook은 컴포넌트를 재사용 가능하게 하는 기반 역할을 한다는 것이다. 더 나아가, '마이크로 상태 관리'는 상태 관리를 더욱 경량화하면서 기존 중앙 집중적인 방식에서 목적 지향적이다.

목적 지향적이라는 말이 포괄적이나, 단순하게 생각하면 책의 예시처럼 UI 컴포넌트로부터 `useCount`라는 hook을 추출해 count 관련 목적을 가진 로직이라는 것을 알 수 있다.

### react hook으로 특정 목적에 따라 다른 해결책 (예시)

1. **폼(Form) 상태는 전역 상태와 별도로 처리해야 하며, 단일 상태로 해결할 수 없다** → 폼 상태는 사용자 입력에 따라 변하는 '일시적인' 데이터로, 다른 컴포넌트와 공유할 필요가 없는 경우가 많다. 전역 상태로 관리하면 불필요한 상태 업데이트와 복잡성이 증가할 수 있다. 따라서 각 폼 컴포넌트 내에서 로컬 상태로 관리하는 것이 효율적이다.
2. **서버 캐시 상태는 다른 상태와는 다른 고유한 특성이 있다** → 서버에서 데이터를 가져와 캐싱하는 상태는 데이터의 일관성, 최신성, 재요청(refetching) 등이 중요하다. 일반적인 상태 관리로는 이러한 요구 사항을 충족시키기 어렵다. 그래서 React Query 같은 라이브러리를 사용하여 서버 상태를 관리한다.
3. **내비게이션 상태는 원본 상태가 브라우저에 있기 때문에 단일 상태는 적합하지 않다** → 내비게이션 상태는 URL, 브라우저 히스토리 등과 밀접하게 연결되어 있다. 이 상태를 일반적인 React 상태로 관리하면 브라우저의 기본 동작과 충돌이 발생할 수 있다. 따라서 React Router와 같은 라이브러리를 사용하여 브라우저 내비게이션 상태와 애플리케이션 상태를 동기화한다.

위의 사례로 미루어 보았을 때, 일시적, 최신 보장, 브라우저 기본 동작과의 충돌 가능성 등의 속성을 가진 데이터는 전역이 아닌 지역적으로 관리 또는 라이브러리 적용을 고민해보아야 할듯 하다.

### 풍부한 그래픽을 제공하는 애플리케이션은 왜 전역 상태가 많이 필요한가?

그래픽 요소들은 보통 실시간으로 업데이트되며, 사용자 입력, 애니메이션, 렌더링 데이터 등이 서로 긴밀하게 연관되어 있다. 예를 들어, 한 컴포넌트에서 발생한 사용자 입력이 다른 컴포넌트의 애니메이션이나 그래픽 표현에 즉각적인 영향을 미칠 수 있다.

우리 서비스 로직도 지금보다 더 그래픽 요소들이 많아질텐데, 그땐 전역 상태 관리를 더 신경써서 해야겠다. 그렇지 않으면 데이터 동기화 문제가 발생할듯..

<br/>

## 🔖 2. 리액트 훅으로 어떻게 마이크로 상태 관리를 할 수 있는가

### 리액트 훅 사용하기

아래의 예시 코드를 보면, `useCount`가 Component에서 분리됐기에 컴포넌트를 건드리지 않고도 기능을 추가할 수 있게 되었다.

```jsx
const useCount = () => {
  const [count, setCount] = useState(0);
  return [count, setCount];
};

const Component = () => {
  const [count, setCount] = useCount();
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
    </div>
  );
};
```

요즘 리팩토링하고 있는 회사 코드에서도 custom hook으로 만들만한 것들을 봐야겠다. 다만, 재사용성을 고려하고 과도한 추상화를 하지 않도록 주의해야겠다.

### custom hook? util?

```jsx
export default function StorageContextProvider({ children }) {
  // ... 상태와 함수들 정의

  const handleUpload = async (files) => {
    // 업로드 로직 구현
  };

  const handleDrop = async (files) => {
    // 드래그 앤 드롭 로직 구현
  };

  return (
    <StorageContext.Provider value={{ /* ... */ }}>
      {children}
    </StorageContext.Provider>
  );
}
```

만약 `handleUpload`와 `handleDrop`를 별도의 hook으로 분리한다? 어차피 이 컴포넌트에서만 사용되는 특정 로직이라면 재사용성은 높아지지 않으면서 코드 구조만 복잡해진다. 그냥 컴포넌트 내부에 유지하는 게 더 유지보수하기 편하다고 볼 수 있다.

반대로, 두 로직이 다른 곳에서도 쓰일 예정이며 파일 업로드 과정에서 상태를 관리한다면 위와 같이 custom hook으로 분리하는 게 적합할 것이다.

```jsx
import { useState, useEffect } from 'react';

const useFileDrop = (curDir, api) => {
  const [loading, setLoading] = useState(false);
  const [progress, setProgress] = useState({ filename: '', progress: 0 });
  const [fileNdirList, setFileNdirList] = useState([]);

  const handleDrop = async (files) => {
    setLoading(true);
    // 로직 구현...
    setLoading(false);
  };

  return {
    loading,
    progress,
    fileNdirList,
    handleDrop,
  };
};

export default useFileDrop;
```

결론: 현재 하고 있는 리팩토링의 목표는 context의 의존성을 모두 끊고, 상태 관리보다는 모듈화를 통해 컴포넌트를 관리하기로 했다. 이럴 때는 그냥 util로 관리하는게 나을 것으로 보인다. 휴 ~

### react hook 함수가 '순수' 해야 한다고?

책 내용 중에 "리액트 훅 함수가 여러번 호출되더라도 일관되게 동작할 수 있도록 '충분히' 순수해야 한다" 라는 부분을 읽고 처음엔 읭? 했다.

차근차근 이해하니 순수 함수의 그 순수라기 보다, React hook은 상태 관리와 side effect 처리를 위해 설계되었기에 완벽한 순수 함수는 아니고 (애초에 그럴 수 없고), **예측 가능하고 일관된 동작을 보장**하는데에 포인트가 있다.

<br/>

## 🔖 3. useState 사용하기

### Lazy Evaluation

```jsx
const heavyComputation = () => {
  // 복잡하고 시간이 오래 걸리는 계산
  let sum = 0;
  for (let i = 0; i < 1000000000; i++) {
    sum += i;
  }
  return sum;
};

const Component = () => {
  const [value, setValue] = useState(heavyComputation);
  // ...
};
```

지연 초기화(Lazy Evaluation)에 대한 책의 설명이 이해가 어려워 더 풀어서 설명해 보겠다.

위 코드에서 `heavyComputation`은 시간이 오래 걸리는 함수다. `useState`에 이 함수를 초기화 함수로 전달하면 컴포넌트가 처음 렌더링될 때만 이 함수가 호출되어 상태의 초기값을 계산한다. 이후 컴포넌트가 다시 렌더링될 때는 이 함수를 다시 호출하지 않으므로 성능에 이점이 있다.

쉽게 말하자면, 필요한 시점에 계산되는 것이다. 첫 렌더링 될 때와 매 랜더링 될 때마다 호출 - 두 개를 비교하자면 아래와 같다.

```jsx
// 초기화 함수로 전달 (첫 렌더링 시 한 번만 호출)
const [state, setState] = useState(() => heavyComputation());

// 초기 상태로 함수 결과를 전달 (매 렌더링마다 호출됨)
const [state, setState] = useState(heavyComputation());
```

<br/>

## 🔖 4. useReducer 사용하기

### 액션에 객체가 없어도 된다?

```jsx
const reducer = (count, delta) => {
  if (delta < 0) {
    throw new Error('delta cannot be negative');
  }
  if (delta > 10) {
    // too big, just ignore
    return count
  }
  if (count < 100) {
    // add bonus
    return count + delta + 10
  }
  return count + delta
}
```

`useReducer` 훅에서 action은 일반적으로 객체로 표현되지만, 반드시 객체일 필요는 없다. 즉, 액션은 어떤 타입의 값이라도 될 수 있다. 예시 코드의 `delta`는 action으로 사용되는 값이며, 숫자 타입(원시 값)이다.

즉, 간단한 상태 업데이트 로직에서는 액션을 숫자나 문자열과 같은 단순한 값으로(도) 사용할 수 있다는 이야기다. 복잡한 상태 관리나 액션 종류를 구분해야 하는 경우엔 액션을 객체로 만들고 `type` 필드를 사용하는 것이 일반적이고.

### 왜 'action' 인가?

action은 state를 어떻게 변경할지 설명하는 정보를 담고 있다. reducer 함수는 `(state, action) => newState` 형태로, 현재 state와 action을 받아 새로운 상태를 반환한다. literally, state를 newState로 반환하는 action이고 상태 변경을 지시하는 역할.

<br/>

## 🔖 5. useState vs. useReducer

### 서로가 서로를 구현할 수 있나

`useReducer`로 `useState`를 100% 구현 가능하다.

```jsx
const useState = (initialState) => {
  const [state, dispatch] = useReducer(
    (prev, action) =>
      typeof action === 'function' ? action(prev) : action,
    initialState,
  );
  return [state, dispatch];
};
```

반대로 `useState`는 `useReducer`를 '거의' 구현 가능하다.

```jsx
const useReducer = (reducer, initialState) => {
  const [state, setState] = useState(initialState);
  const dispatch = (action) =>
    setState(prev => reducer(prev, action));
  return [state, dispatch];
};
```

> [!NOTE]
> `useState` 를 사용한 경우: 컴포넌트 `내부`에 state 업데이트 로직이 존재 <br/>
> `useReducer` 를 사용한 경우: 컴포넌트 `외부`에 state 업데이트 로직이 존재 <br/>

덧붙이자면, `useReducer`의 포인트는 **한 컴포넌트 내 state를 업데이트하는 로직을 해당 컴포넌트로부터 분리**한다.→ 결국 컴포넌트 최적화에 기여한다는 것을 기억하자.

<br/>

## 📚 References

- [React Document - useReducer](https://react.dev/reference/react/useReducer)
- [React Hooks :: useReducer에 대해 알아보기](https://velog.io/@iamhayoung/React-Hooks-useReducer%EC%97%90-%EB%8C%80%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0)
