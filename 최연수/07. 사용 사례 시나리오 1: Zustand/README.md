# 07. 사용 사례 시나리오 1: Zustand

> zustand를 활용한 리렌더링 최적화, 구조화된 데이터 다루기

<br/>

## 🔖 1. Zustand란?

### 특장점

- React 애플리케이션에서 전역 상태를 간단하고 직관적으로 관리할 수 있는 경량 상태 관리 라이브러리
- 복잡한 설정 없이 한 두개의 훅과 간단한 함수 호출만으로 전역 상태를 정의하고 사용할 수 있다.
- store 생성 시, 단순히 객체 형태로 상태와 상태를 변경하는 액션을 정의하면 된다.
- 필요한 상태만 구독하고, 해당 상태가 변경될 때만 리렌더링하므로 불필요한 렌더링을 줄여준다.
- 비동기 로직 (API 통신, 데이터 가져오기 등)을 상태 정의 로직 안에 쉽게 포함할 수 있어 side effect를 처리하기 용이하다.

### 예시 코드

> [Zustand 공식 문서 메인에 있는 코드](https://zustand-demo.pmnd.rs/) 응용

```jsx
import create from 'zustand';

// 1. 전역 상태(스토어) 정의
// create 함수로 상태와 액션(증가, 감소, 리셋)을 정의한 스토어(useStore)를 만든다.
const useStore = create((set) => ({
  count: 0,
  increase: () => set((state) => ({ count: state.count + 1 })),
  decrease: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// 상태 변경 함수인 set을 통해 count 값을 업데이트하면
// count에 의존하는 컴포넌트만 리렌더링 된다.

function Counter() {
  // 2. 상태 사용
  // useStore 훅을 컴포넌트 내에서 호출하면 count와 각 함수를 얻을 수 있다.
  const { count, increase, decrease, reset } = useStore();

  return (
    <div>
      <h1>{count}</h1>
      <button onClick={increase}>곰 한마리 탄생!!</button>
      <button onClick={decrease}>곰 한마리 사망..</button>
      <button onClick={reset}>리셋</button>
    </div>
  );
}

export default function App() {
  return (
    <div>
      <h2>Zustand Bear 🐻</h2>
      <Counter />
    </div>
  );
}
```

### Zustand 쓸까 말까

- 이렇듯, Zustand는 store를 정의하고, hook을 통해 필요한 상태와 함수를 가져와 쓴다.
- 전역 상태가 한번에 정의되고 관리되기에 상태 변경 로직을 한 곳에 모을 수 있어 규모가 커져도 유지보수가 쉬울 것으로 기대된다.
- 따라서 한번에 전역 상태 구조를 명확히 잡고 하나의 store를 통해 상태 관리 흐름을 단순화하고 싶다면 Zustand가 좋겠다.
- Jotai는 처음부터 상태를 atom 단위로 쪼개야 해서 프로젝트 초기 단계에서 잘 사용할 수 있을지 모르겠다.

<br/>

## 🔖 2. 불변 상태 모델

### '불변 상태 모델'을 풀어 쓰기

- Zustand는 상태를 관리할 때, 원래 상태 객체를 바꾸는 대신 변경된 부분만 새로 만든 객체에 담아 상태를 업데이트한다.
- 즉, 상태 변경 시 기존 객체를 수정하는 게 아니라, 새로운 객체를 만들어 그 객체를 기존 상태 대신 넣어준다.
- 그리고 변경하지 않은 부분 (e.g. 다른 필드나 내부 객체)은 그대로 이전 것을 재사용해 불필요한 렌더링을 줄일 수 있다.
- 비유하자면, 종이에 쓰인 내용을 지우개로 지우고 다시 쓰는 게 아니라, 새로운 종이에 바뀐 내용만 써서 교체한다. 바뀌지 않은 내용은 예전 내용을 그대로 가져와서 리소스를 낭비하지 않을 수 있다.

### 예시 코드

```jsx
import create from 'zustand';

const useStore = create((set) => ({
  user: { name: 'Ella', age: 20 },
  // 상태 변경 함수 (불변성 유지)
  updateUserName: (newName) => set((state) => ({
    // user 객체를 직접 수정하지 않고, 새로운 객체를 return
    // name만 바뀌고 age는 바뀌지 않았으니 기존 user.age를 그대로 재사용
    user: { ...state.user, name: newName }
  })),

  items: ['apple', 'banana'],
  addItem: (item) => set((state) => ({
    // 기존 배열을 수정하지 않고, 새 배열을 만들어 return
    items: [...state.items, item]
  }))
}));

function App() {
  const { user, updateUserName, items, addItem } = useStore();

  return (
    <div>
      <p>User: {user.name}, {user.age}</p>
      <button onClick={() => updateUserName('Ella')}>이름을 Chloe로 바꾸셈</button>

      <p>Items: {items.join(', ')}</p>
      <button onClick={() => addItem('orange')}>Orange도 주셈</button>
    </div>
  );
}

export default App;
```

### 예시를 통해 장점 느껴보기

- 이렇듯 Zustand는 상태가 어떻게 바뀌었는지 쉽게 파악할 수 있다.
- 새로운 객체를 반환하는 불변성 패턴을 활용하기 때문에, 이전 상태와 새로운 상태를 비교하는 과정이 단순해지는 것이다.
- 책에서 말하는 "상태 객체의 참조에 대한 동등성만 확인하면 변경 여부를 알 수 있으므로 객체의 값 전체를 확인할 필요가 없다" 이 문장을 풀어 이야기하면, “같은 객체냐, 다른 객체냐”를 확인하는 것만으로 상태 변경 여부를 판단할 수 있다는 것이다.

<br/>

## 🔖 3. 수동 렌더링 최적화

### 선택자 기반 리렌더링 제어란?

- 컴포넌트가 필요로 하는 특정 부분의 상태만 subscribe하고, 그 부분이 실제로 변경되었을 때만 컴포넌트를 다시 렌더링하는 전략이다.
- 이렇게 하면 전역 상태의 다른 부분이 변해도 컴포넌트는 영향을 받지 않아, 불필요한 리렌더링을 줄일 수 있다.
- 큰 전역 상태 중 아주 일부분만 사용하는 컴포넌트는 그 일부분이 변할 때만 렌더링되기에, 성능 최적화에 유리할 것이다.

### 예시 코드

```jsx
import create from 'zustand';

const useStore = create((set) => ({
  user: { name: 'Ella', age: 20 },
  items: ['apple', 'banana'],
  setName: (name) => set((state) => ({
    user: { ...state.user, name }
  })),
  addItem: (item) => set((state) => ({
    items: [...state.items, item]
  })),
}));

// 선택자: user의 name만 가져오기
const selectUserName = (state) => state.user.name;

// 선택자: items 배열의 길이만 가져오기
const selectItemsCount = (state) => state.items.length;

function UserNameDisplay() {
  // user.name 값만 구독, name이 바뀌지 않으면 리렌더링 안 함
  const name = useStore(selectUserName);
  console.log('UserNameDisplay 렌더링한당');
  return <div>User Name: {name}</div>;
}

function ItemsCountDisplay() {
  // items.length 값만 구독, items 개수가 바뀌지 않으면 리렌더링 안 함
  const count = useStore(selectItemsCount);
  console.log('ItemsCountDisplay 렌더링한당');
  return <div>Items Count: {count}</div>;
}

function App() {
  const setName = useStore((state) => state.setName);
  const addItem = useStore((state) => state.addItem);

  return (
    <div>
      <UserNameDisplay />
      <ItemsCountDisplay />
      <button onClick={() => setName('Bob')}>이름을 Chloe로 바꾸셈</button>
      <button onClick={() => addItem('orange')}>Orange도 주셈</button>
    </div>
  );
}

export default App;
```
