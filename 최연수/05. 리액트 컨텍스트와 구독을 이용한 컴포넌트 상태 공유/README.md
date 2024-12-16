# 05. 리액트 컨텍스트와 구독을 이용한 컴포넌트 상태 공유

> 모듈 상태의 한계, 컨텍스트 사용이 필요한 시점, 컨텍스트와 구독 패턴

<br/>

## 🔖 1. Context, Props, Subscription 비교

### Context

트리 구조 하위에 있는 컴포넌트들이 중간 컴포넌트들에 Props를 일일이 거치지 않고도 필요한 값에 접근한다. 또한 하나의 Context들에 대해 서로 다른 provider를 사용하면, 각 provider가 감싸고 있는 하위 트리마다 서로 다른 값을 제공할 수 있다.

### Props

사실 Props만으로도 상위에서 하위 트리로 내려가는 과정에 각기 다른 값을 줄 수 있다. 하지만 이 경우 하위로 내려가는 모~든 경로마다 Props를 계속 전달(Props drilling)해야 하므로 깊은 트리 구조에서는 관리가 어려워지며 가독성 측면에서 좋지 않다.

### Subscription

구독 패턴 혹은 `useSubscription`을 활용하면 변경된 상태를 필요로하는 컴포넌트들만 리렌더링하게 하여 불필요한 리렌더링을 줄일 수 있다. 예를 들어, 전역 상태 괸리나 컨텍스트 값을 직접 구독하고, 필요한 부분만 리렌더링하게 한다면 상태 변경과 무관한 컴포넌트까지 다시 렌더링 되는 상황을 피할 수 있다.

### 프로젝트에 어떤 것을 적용할까?

현재 프로젝트에서는 Context를 주로 사용하다가 Props로 빼는 상황이었다. 그러다 지난 주, 리팩토링을 하면 할수록 모든 상태를 100% prop으로 관리하기 어렵다는 것을 깨달았다. 따라서 아래와 같은 방향으로 진행하고자 한다.

> [!NOTE]
>
> 1. 여러 컴포넌트가 동일한 상태를 필요로 한다면, 해당 상태는 Context로 관리한다.
> 2. 전역으로 사용하지 않아도 되는 상태는 각 컴포넌트에서 `useState`나 `useReducer` 등을 활용한다. 이를 통해 각 상태를 독립적으로 관리함으로써 상태의 책임 범위를 명확히 하고, 로컬화 한다.
> 3. 일부 컴포넌트가 전역 상태 변경에 지나치게 자주 렌더링된다면, `useSubscription`을 통해 필요한 컴포넌트만 업데이트 되도록 한다. 이는 성능 문제가 실제로 발생했을 때 대응하는 전략으로 채택한다.

<br/>

## 🔖 2. 모듈 상태의 한계

### 모듈 상태 쉽게 이해하기

> 모듈 상태는 리액트 컴포넌트 외부에 존재하는 전역으로 정의된 싱글턴이기 때문에 컴포넌트 트리나 하위 트리마다 다른 상태를 가질 수 없다는 한계가 있다.

- "모듈 상태"는 컴포넌트나 컨텍스트처럼 여러 개를 만들 수 있는 게 아니라, 프로젝트 전역적으로 딱 한 번 정의되고 공유되는 변수(싱글턴)다.
- 예를 들어, 어떤 전역 변수를 하나 정해놓으면 그 값은 앱 어디서나 동일하게 접근된다. 따라서 컴포넌트 트리나 하위 트리별로 그 변수를 다르게 가질 수 없다.
- 쉽게 말해, "모듈 상태"는 온 집안 식구가 공유하는 한 개의 TV 같은 것이다. 집 안(앱) 어느 방(컴포넌트 트리)에서나 같은 채널(상태)을 보고 있어야 하고, 방마다 채널을 다르게 틀 수 없는 것!
- 따라서 책에서 언급한듯이 모듈 상태는 하위 트리마다 다른 상태를 가질 수 없다는 한계가 있다.

<br/>

## 🔖 3. 컨텍스트 사용이 필요한 시점

### 동일한 컴포넌트를 연속해서 사용하는 사례

> 동일한 컴포넌트를 연속해서 사용하는 상황은 실제로 어떤 상황에 적용할 수 있을까? 이런 경험이 별로 없던 것 같아서 찾아보았다.

- **반복되는 UI 패턴**: 현재 로그인한 유저의 정보를 마이 페이지 상단과 사이드 바 등 여러 구역에서 보여주고 싶을 때도 유용할 것 같다.
- 책에서 언급하듯이 (p. 101) 만약 세 컴포넌트가 provider가 다른 것을 제외하고 코드가 거의 동일하다면, 이것이 바로 리액트 컨텍스트를 사용하기에 적합한 곳이라고 했다. 일종의 signal로 보아도 좋을듯.
- **목록이나 카드 UI**: 전역 상태로 관리되는 정렬 옵션, 필터 정보가 있다 치자. 이 상태를 참조하는 카드 컴포넌트가 있고, 이 카드 컴포넌트를 화면에 여러 개 나열한다 치자. 모두 같은 전역 상태를 참고하지만 UI는 동일한 컴포넌트를 연속해서 렌더링할 때 사용하면 좋을듯 하다.

### 동일한 컴포넌트를 연속해서 사용하는 예시 코드

전역 상태(정렬 옵션, 필터 정보)를 컨텍스트로 관리하고, 각 카드 컴포넌트에서 `useContext`와 `useSubscription`을 통해 필요한 부분만 구독하여 렌더링하는 간단한 예시 코드를 작성해 보았다.

```jsx
import React, { createContext, useContext, useMemo, useState } from 'react';
import { useSubscription } from 'use-subscription';

// 전역 상태 관리용 store를 생성하는 함수
function createStore(initialState) {
  let state = initialState; // 상태를 저장할 변수
  const listeners = new Set(); // 상태 변경 시 호출할 콜백들을 담는 집합

  return {
    // 현재 state를 반환하는 함수
    getState: () => state,
    // state를 업데이트하는 함수 (객체 병합 후 모든 리스너에게 알림)
    setState: (newState) => {
      state = { ...state, ...newState };
      listeners.forEach((listener) => listener());
    },
    // 리스너(콜백) 등록 함수, 상태가 변경되면 콜백 실행
    subscribe: (listener) => {
      listeners.add(listener);
      // 구독 해제를 위한 함수 반환
      return () => listeners.delete(listener);
    },
  };
}

// 초기 상태: 필터를 'ALL'로 설정
const store = createStore({ filter: 'ALL' });

// store를 제공하기 위한 컨텍스트 생성
const StoreContext = createContext(store);

// 필터 상태를 구독하고 가져오는 useFilter 훅
function useFilter() {
  const store = useContext(StoreContext); // 컨텍스트에서 store 가져오기
  const subscription = useMemo(
    () => ({
      // 현재 필터 상태를 반환하는 함수
      getCurrentValue: () => store.getState().filter,
      // 상태 변경 시 호출할 콜백 등록 함수
      subscribe: (callback) => store.subscribe(callback),
    }),
    [store] // store가 변경될 때만 재생성
  );
  return useSubscription(subscription); // useSubscription으로 필터 상태 변화에 반응
}

// 필터 상태를 변경하는 함수를 반환하는 훅
function useSetFilter() {
  const store = useContext(StoreContext); // 컨텍스트에서 store 가져오기
  // 새로운 필터 값을 설정하는 함수 반환
  return (newFilter) => store.setState({ filter: newFilter });
}

// 아이템 정보를 받아와서 필터 조건에 맞는지 판단하고 렌더링하는 컴포넌트
function Card({ item }) {
  const currentFilter = useFilter(); // 위에서 만든 hook으로 현재 필터 가져오기
  const visible = currentFilter === 'ALL' || item.type === currentFilter; // 현재 아이템 타입이 필터 조건에 맞는지 확인

  // 필터 조건에 맞으면 아이템을 보여주고, 아니면 null 반환
  return visible ? <div style={{ border: '1px solid #333', margin: '4px', padding: '8px' }}>{item.name}</div> : null;
}

// 필터를 변경할 수 있는 버튼들을 가진 컴포넌트
function FilterControls() {
  const setFilter = useSetFilter(); // 필터 변경 함수 가져오기
  return (
    <div>
      <button onClick={() => setFilter('ALL')}>ALL</button> {/* ALL 필터로 변경하는 버튼 */}
      <button onClick={() => setFilter('BOOK')}>BOOK</button> {/* BOOK 필터로 변경하는 버튼 */}
      <button onClick={() => setFilter('TOY')}>TOY</button> {/* TOY 필터로 변경하는 버튼 */}
    </div>
  );
}

// 예시 아이템 배열 (type: BOOK 또는 TOY)
const items = [
  { name: 'Item 1', type: 'BOOK' },
  { name: 'Item 2', type: 'TOY' },
  { name: 'Item 3', type: 'BOOK' },
];

export default function App() {
  return (
    // StoreContext.Provider로 감싸 전역 상태를 제공
    <StoreContext.Provider value={store}>
      <h1>Cards with global filter</h1>
      <FilterControls /> {/* 필터 변경 컴포넌트 */}
      <div style={{ display: 'flex' }}>
        {/* items 배열을 순회하며 Card 컴포넌트 렌더링 */}
        {items.map((item, index) => (
          <Card key={index} item={item} />
        ))}
      </div>
    </StoreContext.Provider>
  );
}
```

<br/>

## 🔖 4. useSubscription 더 파고들기

> [React가 `useSubscription` hook을 사용한 예시 코드](https://github.com/facebook/react/blob/7283a213dbbc31029e65005276f12202558558fc/packages/react-devtools-shared/src/devtools/views/Settings/SettingsModalContextToggle.js#L16)

React 코드에서는 `useSubscription` 훅을 외부 상태의 변경 사항에 반응하여, 컴포넌트를 재렌더링하기 위한 구독 매커니즘을 구현하는 데 사용되었다.

### (1) 구독 객체 생성

```jsx
const isProfilingSubscription = useMemo(() => ({
  getCurrentValue: () => profilerStore.isProfilingBasedOnUserInput,
  subscribe: (callback: Function) => {
    profilerStore.addListener('isProfiling', callback);
    return () => profilerStore.removeListener('isProfiling', callback);
  },
}), [profilerStore]);
```

먼저 `useMemo`를 사용하여 `isProfilingSubscription`이라는 구독 객체를 만든다. 이 객체는 두 가지 주요 속성을 가진다.

- `getCurrentValue`: 현재 `profilerStore`에서 필요한 값을 반환하는 함수
- `subscribe`: 값이 변경될 때마다 호출될 콜백 함수를 등록하는 함수와 등록 취소를 위한 반환 함수

여기서 `profilerStore.addListener('isProfiling', callback)`를 통해 `profilerStore`의 `isProfiling` 상태가 변경될 때마다 `callback`을 호출하도록 구독을 설정하고, `return` 부분에서 해당 구독을 해제하는 함수도 제공한다.

### (2) useSubscription으로 구독 연결

`useSubscription` 훅에 이 구독 객체 `isProfilingSubscription`을 전달하면, 컴포넌트는 `profilerStore.isProfilingBasedOnUserInput` 값에 대한 "실시간 반응"을 가지게 된다.

```jsx
const isProfiling = useSubscription<boolean>(isProfilingSubscription);
```

`useSubscription`은 내부적으로 `subscribe` 함수를 통해 외부 상태 변화를 감지하고, 값이 변경되면 컴포넌트를 재렌더링하여 `isProfiling` 값을 최신 상태로 유지한다.

### (3) 최신 값 사용

이렇게 얻어진 `isProfiling` 값은 아래 `Button` 컴포넌트의 `disabled` 속성에 반영되며, `profilerStore`의 상태 변화(즉, `isProfiling` 상태 변화)에 따라 버튼이 활성/비활성화 된다.

```jsx
<Button
  disabled={isProfiling}
  onClick={showFilterModal}
  title="View settings">
  <ButtonIcon type="settings" />
</Button>
```

### 정리하자면

`useSubscription`을 사용하여 `profilerStore`라는 외부 상태와 컴포넌트를 연결하고, 해당 상태가 바뀔 때마다 자동으로 컴포넌트를 업데이트하는 구독 로직을 간결하게 구현한 것이다.
