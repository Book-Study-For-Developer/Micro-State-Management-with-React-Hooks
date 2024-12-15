## 06. 전역 상태 관리 라이브러리 소개

### 전역 상태 관리 문제 해결하기 
전역 상태를 설계할때는 두 가지 문제점이 있다.

- 첫번째 문제점은 **전역 상태를 읽는 방법이다.** 
  - 전역 상태는 여러 값을 가질 수 있고, 사용하는 컴포넌트는 모든 값이 필요하지 않을 수 있다. 또한 전역 상태가 바뀌면 리렌더링이 발생하는데, 변경된 값이 컴포넌트와 연관이 없을 경우에도 리렌더링이 발생한다.
- 두번째 문제점은 **전역 상태에 값을 넣거나 갱신하는 방법이다.**
  - 전역 변수에서 하나의 프로퍼티를 개발자가 직접 값을 변경하는 예

위와 같은 문제를 해결하기 위해서는 변경하는 함수를 제공해야한다.

```jsx
const createContainer = () => {
  let state = { a: 1, b: 2};
  const getState = () => state;
  const setState = () => {...};
  return {getState , setState};
}

const globalContainer = createContainer();
globalContainer.setState(...);
```

### 리렌더링 최적화 
전역 상태에서 리렌더링을 피하는 것은 정말 중요한 문제이다. 

```js
let state = {
  a: 1,
  b: { c:2, d:3 },
  e: { f:4, g:5 }
}
```
위와 같은 전역상태 state가 있을때 a의 값을 변경하더라도 b와 e의 특정 값을 사용하는 다른 컴포넌트에서는 리렌더링이 발생해서는 안된다. 

위와 같은 점을 해결하기 위해서는 컴포넌트에서 state의 어느 부분이 사용될지 지정하는것이다.
대표적으로는 3가지 방법이 있다.

- 선택자 함수 사용
- 속성 접근 감지
- 아톰 사용

### 선택자 함수 사용
선택자 함수를 이용해서 상태를 받아 상태의 일부를 반환하는 것이다. 

```jsx
const Component = () => {
  const value = useSelector((state) => state.b.c);
  return <>{value}</>;
}
```

useSelector와 같은 선택자 함수를 활용해서 해당 컴포넌트에서 사용하지 않는 특정값이 변경됐더라도 리렌더링을 발생시키지 않아야 한다.

### 속성 접근 감지

```jsx
const Component = () => {
  const trackedState = useTrackedState();
  return <p>{trackedState.b.c}</p>
}
```

useTrackedState hook을 사용해서 자동으로 속성에 접근했음을 감지하고 b.c의 값이 변경될때만 리렌더링을 발생시킨다. 

생전 처음 들어보는 hook이라 실제로 있는지 확인해봤더니 있었고 이 책의 글쓴이가 maintainer로 있는 react-tracked 라이브러리이다. 

책에서는 너무 간단하게만 살펴봤기에 내부 내용을 조금 더 살펴봤다. 

```js
export const createTrackedSelector = <State>(
  useSelector: <Selected>(selector: (state: State) => Selected) => Selected,
) => {
  const useTrackedSelector = () => {
    const [, forceUpdate] = useReducer((c) => c + 1, 0);
    // per-hook affected, it's not ideal but memo compatible
    const affected = useMemo(() => new WeakMap(), []);
    const prevState = useRef<State>();
    const lastState = useRef<State>();
    useEffect(() => {
      if (
        prevState.current !== lastState.current &&
        isChanged(prevState.current, lastState.current, affected, new WeakMap())
      ) {
        prevState.current = lastState.current;
        forceUpdate();
      }
    });
    const selector = useCallback(
      (nextState: State) => {
        lastState.current = nextState;
        if (
          prevState.current &&
          prevState.current !== nextState &&
          !isChanged(prevState.current, nextState, affected, new WeakMap())
        ) {
          // not changed
          return prevState.current;
        }
        prevState.current = nextState;
        return nextState;
      },
      [affected],
    );
    const state = useSelector(selector);
    if (hasGlobalProcess && process.env.NODE_ENV !== 'production') {
      // eslint-disable-next-line react-hooks/rules-of-hooks
      useAffectedDebugValue(state, affected);
    }
    const proxyCache = useMemo(() => new WeakMap(), []); // per-hook proxyCache
    return createProxy(state, affected, proxyCache);
  };
  return useTrackedSelector;
};
```


**createTrackedSelector**
```js
export const createTrackedSelector = <State>(
  useSelector: <Selected>(selector: (state: State) => Selected) => Selected,
)
```

- 상태 변화를 추적할 수 있는 새로운 셀렉터를 생성하는 함수이다.

**내부 구현의 주요 상태들**
```js
const useTrackedSelector = () => {
  const [, forceUpdate] = useReducer((c) => c + 1, 0);  // 컴포넌트 강제 리렌더링용
  const affected = useMemo(() => new WeakMap(), []);    // 변경된 부분 추적
  const prevState = useRef<State>();                     // 이전 상태 저장
  const lastState = useRef<State>();                     // 마지막 상태 저장
```

**상태 변화 감지**

```js
useEffect(() => {
  if (
    prevState.current !== lastState.current &&
    isChanged(prevState.current, lastState.current, affected, new WeakMap())
  ) {
    prevState.current = lastState.current;
    forceUpdate();  // 변화가 감지되면 리렌더링
  }
});
```

**셀렉터 로직**

```js
const selector = useCallback(
  (nextState: State) => {
    lastState.current = nextState;
    // 상태가 실제로 변경되었는지 확인
    if (
      prevState.current &&
      prevState.current !== nextState &&
      !isChanged(prevState.current, nextState, affected, new WeakMap())
    ) {
      return prevState.current;  // 변경되지 않았다면 이전 상태 반환
    }
    prevState.current = nextState;
    return nextState;  // 변경되었다면 새 상태 반환
  },
  [affected],
);
```

**프록시 생성 및 반환**
```js
const proxyCache = useMemo(() => new WeakMap(), []); 
return createProxy(state, affected, proxyCache);
```

### 아톰 사용
아톰은 리렌더링을 발생시키는 데 필요한 최소 상태 단위이다. 전체 전역 상태를 구독해서 리렌더링을 피하는 대신 아톰을 사용하면 좀 더 세분화해서 구독하는 것이 가능하다.

```js
const globalState = {
  a: atom(1),
  b: atom(2),
  e: atom(3),
};

const Component = () => {
  const value = useAtom(globalState.a);
  return <>{value}</>
}
```

아톰을 사용하는 접근 방식은 수동 최적화와 자동 최적화의 중간 정도로 볼 수 있다. 아톰과 파생 값의 정의는 명시적이지만 의존성 추적은 자동으로 된다.
