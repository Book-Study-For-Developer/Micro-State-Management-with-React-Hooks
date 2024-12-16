## 시나리오 1: Zustand

Zustand는 주로 리액트 모듈 상태를 생성하도록 설계된 작은 라이브러리이다. 상태 객체를 수정할 수 없고 항상 새로 만들어야 하는 불변 갱신 모델을 기반으로 한다.

### 모듈 상태와 불변 상태 이해하기 

Zustand는 상태를 변경하기 위해서는 새 객체를 생성해서 대체해야 하며, 수정하지 않은 객체는 재사용해야 한다. 불변 상태의 장점은 상태 객체의 참조에 대한 동등성만 확인하면 변경 여부를 알 수 있다.

```js
const store = create(() => ({count : 0}));

console.log(store.getState()); // {count : 0}
store.setState({count : 1});
console.log(store.getState()); // {count : 1}
```

여러개의 속성을 가질때의 예제를 아래와 같이 살펴보자 

```js
const store = create(() => ({
  count: 0,
  text: 'hello'
}));

store.setState({
  count: 1,
  text: 'hello'
});

consoe.log(store.getState()); // {count:1, text:'hello'}
store.setState({
  count: 2,
});
console.log(store.getState()); // {count:2, text:'hello'}
```

### 리액트 훅을 이용한 리렌더링 최적화

전역 상태를 사용하는 경우 모든 컴포넌트가 전역 상태를 사용하는것은 아니기 때문에 리렌더링 최적화가 필요하다.

```js
const useStore = create(() => ({
  count : 0,
  text: 'hello',
}));

const Component = () => {
  const {count , text} = useStore();
  return <div>count : {count}</div>;
}
```

위와 같이 사용하게 된다면 화면과 상관없는 text값이 변경되면 리렌더링이 발생하는데 이를 해결하기 위해서 선택자 함수를 지정할 수 있다.

```js
const Component = () => {
  const count = useStore((state) => state.count);
  return <div>count : {count}</div>;
}
```
위와 같은 선택자 기반 리렌더링 제어를 **수동 렌더링 최적화**라고 한다.

### 리액트 생명주기에서 사용하는 비모듈 상태 사용 예제

챕터의 마지막장에서 리액트 생명주기에서 store를 생성하는 비모듈상태란 무엇인가에 대해서 한번 알아봤다. 

```js
import { createStore } from 'zustand/vanilla' // vanilla를 사용하는 것에 주목

const createBearStore = (initialCount = 0) => {
  return createStore((set) => ({
    bears: initialCount,
    addBear: () => set((state) => ({ bears: state.bears + 1 })),
    removeBear: () => set((state) => ({ bears: state.bears - 1 })),
  }))
}
```

위와 같이 vanilla를 사용해서 store를 생성한다. 


```js
import { useState, useEffect } from 'react'
import { useStore } from 'zustand'

function useBearStore(initialCount) {
  // store 인스턴스 상태 관리
  const [store, setStore] = useState(null)

  useEffect(() => {
    // 컴포넌트 마운트 시 store 생성
    const newStore = createBearStore(initialCount)
    setStore(newStore)

    // 클린업 함수
    return () => {
      // 컴포넌트 언마운트 시 store 정리
      newStore.destroy()
    }
  }, [initialCount])

  // store가 없으면 null 반환
  if (!store) return null

  // store 상태와 액션 반환
  return {
    bears: store.getState().bears,
    addBear: store.getState().addBear,
    removeBear: store.getState().removeBear,
  }
}
```









