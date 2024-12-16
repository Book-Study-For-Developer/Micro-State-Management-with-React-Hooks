## 07. 사용 사례 시나리오 1: Zustand

> [!NOTE]
> Zustand는 리액트의 모듈 상태를 생성하도록 설계된 '작은' 라이브러리다. **상태 객체를 수정할 수 없고 항상 새로 만들어야 하는 불변 갱신 모델을 기반으로 한다.**

불변 상태의 장점은

- 상태 객체의 참조에 대한 \*동등성만 확인하면 변경 여부를 알 수 있으므로 객체의 값 전체를 확인할 필요가 없다. (📌 객체의 중첩 정도가 깊은 경우에 비교하는 데 그만큼 비용을 지불해야 한다...!)

불변 상태를 사용하는 방식은 다음과 같다.

```tsx
// store.ts
import create from "zustand";

export const store = create(() => ({ count: 0 }));

// 적절한 사용
console.log(store.getState()); // { count : 0 }
store.setState({ count: 1 }); // 참조값이 다른 새로운 객체 전달
console.log(store.getState()); // { count : 1 }

// 부적절한 사용
// state1은 이전 상태와 동일한 참조를 가지기 때문에 라이브러리는 변경 사항 감지 X
const state1 = store.getState();
state1.count = 2;
store.setState(state1);
```

> [!NOTE]
> store 객체에서 제공하는 API인 setState()는 새 상태와 이전 상태를 병합한다. 내부적으로 Object.assign()이 구현되어 있다. `Object.assign({}, oldState, newState);`

> [!NOTE]
> subscribe()에 상태가 변경될 때마다 호출되는 콜백 함수를 전달할 수 있다. setState()를 사용하면 콜백 함수가 실행된다.

### 리액트 훅을 이용한 리렌더링 최적화 - 선택자 함수를 지정하여 원하는 상태만 리렌더링

```tsx
// store.ts
import create from "zustand";

export const useStore = create(() => ({
  count: 0,
  text: "hello",
}));

// Component.tsx
import { useStore } from "./store.ts";

const Component = () => {
  const { count, text } = useStore();
  return <div>count: {count}</div>;
};
```

위 컴포넌트의 문제는 화면과 관련 없는 text가 변경되더라도 리렌더링이 된다는 것이다. 리렌더링을 피하기 위해서 useStore에 선택자 함수를 지정할 수 있다.

```tsx
const Component = () => {
  const count = useStore((state) => state.count);
  return <div>count: {count}</div>;
};
```

이러한 선택자 기반 리렌더링 제어를 **수동 렌더링 최적화**라고 한다.

### 읽기 상태와 갱신 상태 사용하기

```tsx
type StoreState = {
  count1: number;
  count2: number;
};

const useStore = create<StoreState>(() => ({
  count1: 0,
  count2: 0,
}));

// 선택자 정의
const selectCount1 = (state: StoreState) => state.count;

const Counter1 = () => {
  const count1 = useStore(selectCount1);
  const inc1 = () => {
    useStore.setState(
      (prev) => ({ count1: prev.count1 + 1 });
    );
  };

  return (
    <div>
      count1: {count1} <button onClick={inc1}>+1</button>
    </div>
  );
};
```

inc1 함수는 더 높은 재사용성과 가독성을 위해 store에 함수를 미리 정의할 수 있다.

```tsx
type StoreState = {
  count1: number;
  count2: number;
  inc1: () => void;
  inc2: () => void;
};

// store에 미리 정의한 상태 갱신 함수
const useStore = create<StoreState>((set) => ({
  count1: 0,
  count2: 0,
  inc1: () => set((prev) => ({ count1: prev.count1 + 1 })),
  inc2: () => set((prev) => ({ count2: prev.count2 + 1 })),
}));

const selectCount2 = (state: StoreState) => state.count2;
const selectInc2 = (state: StoreState) => state.inc2;

const Counter2 = () => {
  const count2 = useStore(selectCount2);
  const inc2 = useStore(selectInc2);
  return (
    <div>
      count2: {count2}
      <button onClick={inc2}>+1</button>
    </div>
  );
};
```

상태 갱신 함수(inc1, inc2)를 상태 값과 가깝게 배치해서 Zustand의 setState가 이전 상태와 새로운 상태를 병합할 수 있다.

> [!NOTE]
> 파생 상태를 생성하려면 파생 상태에 대한 선택자를 사용하면 된다.

```tsx
const Total = () => {
  const count1 = useStore(selectCount1);
  const count2 = useStore(selectCount2);

  return <div>total: {count1 + count2}</div>;
};
```

위 코드는 +라는 비즈니스 로직을 통해 합계라는 파생 상태를 만들어냈지만 문제가 있다. count1이 1만큼 감소하고 count2가 1만큼 증가하면 합계는 변한 것이 없지만 리렌더링된다.

해결 방법으로는,

```tsx
// 1. 실제로 합계가 변경될 때만 리렌더링될 수 있도록 더하기를 수행하는 선택자 생성
const selectTotal = (state: StoreState) => state.count1 + state.count2;

const Total = () => {
  const total = useStore(selectTotal);
  return <div>total: {total}</div>;
};

// 2. store 내부에서 합계 생성
const useStore = create((set) => ({
  count1: 0,
  count2: 0,
  total: 0,
  inc1: () =>
    set((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
      total: prev.count1 + 1 + prev.count2, // 근데, 이렇게 하면 계산 인자가 많아지면 모두 다 써줘야 할텐데...
    })),
  inc2: () =>
    set((prev) => ({
      ...prev,
      count2: prev.count2 + 1,
      total: prev.count2 + 1 + prev.count1,
    })),
}));
```

### 구조화된 데이터 처리하기

(Todo 어플리케이션 구현은 생략...)

> [!IMPORTANT]
> store 내에서 todos 상태를 관리하고 있고, 해야 할 일을 todos 배열에 추가하는 경우를 생각해보자. 상태의 참조가 변경되는지를 감지하는 Zustand의 경우는 todos에 새로운 원소가 추가되면 변경을 감지하여 리렌더링이 촉발된다. ⚠️ 상위 컴포넌트가 리렌더링되면 하위 컴포넌트가 모두 리렌더링되는 문제를 해결하기 위해서, todos 하위에 있는 TodoItem 컴포넌트를 `memo` 함수로 감싸면 리렌더링에 사용되는 비용을 줄일 수 있다.

### 이 접근 방식과 라이브러리의 장단점

Zustand의 읽기 및 쓰기 상태에 대해 요약하면,

- 읽기 상태: 리렌더링을 최적화하기 위해 선택자 함수를 사용
- 쓰기 상태: 불변 상태 모델을 기반으로 함

핵심은 **'리액트'**가 최적화를 위해 객체 불변성을 기반으로 한다는 점이다.

```tsx
const countObj = { value: 0 };

const Component = () => {
  const [count, setCount] = useState(countObj);
  const handleClick = () => {
    setCount(countObj);
  };

  useEffect(() => {
    console.log("component updated");
  });

  return (
    <>
      {count.value}
      <button onClick={handleClick}>Update</button>
    </>
  );
};
```

위에 코드에서는 아무리 버튼을 눌러도 useEffect에 전달한 콜백이 실행되지 않는다. 객체 참조 자체가 동일하기 때문에 변경되지 않는다고 추측하기 때문이다.

💡 Zustand 상태 모델은 이러한 객체 불변성 규칙과 완전히 일치하고, 선택자 함수를 사용한 Zustand의 렌더링 최적화 역시 불변성을 기반으로 한다. Zustand는 리액트와 동일한 모델을 사용해서 **라이브러리의 단순성과 번들 크기가 작다는 점**에서 큰 이점이 있다.

npm에서 알아본 바로는,

- Zustand(5.0.2 버전) unpacked size: 88.8kB
- Jotai(2.10.3 버전) unpacked size: 438kB

단점은, 선택자를 이용한 수동 렌더링 최적화다. 객체 참조 동등성을 이해해야 하고, 선택자 코드를 위해 보일러플레이트 코드를 많이 작성해야 할 필요가 있다.
