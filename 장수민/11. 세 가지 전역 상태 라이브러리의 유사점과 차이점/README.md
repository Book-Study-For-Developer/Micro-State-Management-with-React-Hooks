## 11. 세 가지 전역 상태 라이브러리의 유사점과 차이점

> [!IMPORTANT]
> Zustand는 사용법과 스토어 모델 측면에서 Redux(or React Redux)와 유사하지만, 리듀서를 기반으로 하지 않는다는 점에서 다르다.

> [!IMPORTANT]
> Jotai는 API 측면에서 Recoil과 유사하지만 선택자 기반이 아니고 렌더링 최적화를 위한 최소한의 API를 제공하는 것이 목표다.

> [!IMPORTANT]
> Valtio는 변경 가능한 갱신 모델 측면에서 MobX와 조금 유사하지만 렌더링 최적화 구현 방식이 매우 다르다.

### Zustand와 Redux의 차이점

COMMON : 두 라이브러리 모두 단방향 데이터 흐름을 기반으로 한다. 상태를 갱신하라는 명령을 나타내는 action을 실행해서 갱신된 후에 새로운 상태가 필요한 곳으로 전파된다. 📌 디스패치와 전파를 분리하는 것은 데이터의 흐름을 단순화하고 전체 시스템을 더 예측 가능하게 만든다. 📌

> [!NOTE]
> 양방향 데이터 흐름은 어디서 어디로 명령이 전달됐는지 파악하기가 매우 힘들다.

DIFF : Redux는 리듀서(이전 상태와 action 객체를 받아 새로운 상태를 반환하는 순수 함수)에 기반하지만, Zustand는 그럴 필요가 없다.

리덕스는,

```tsx
import { configureStore } from "@reduxjs/toolkit";
import counterReducer from "../features/counter/counterSlice";

// Redux 스토어를 생성하기 위한 configureStore API 사용
export const store = configureStore({
  reducer: {
    counter: counterReducer,
  },
});

// -----------------------------

import { createSlice } from "@reduxjs/toolkit";

const initialState = {
  value: 0,
};

// 리듀서와 액션을 모두 포함한 counterSlice
export const counterSlice = createSlice({
  name: "counter",
  initialState,
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
  },
});

// -----------------------------

import { useSelector, useDispatch } from "react-redux";
import { decrement, increment } from "./counterSlice";

// selector 함수를 사용해서 스토어에서 counter 값을 가져온다.
export function Counter() {
  const count = useSelector(
    (state: { counter: { value: number } }) => state.counter.value
  );
  const dispatch = useDispatch();

  return (
    <div>
      <button onClick={() => dispatch(increment())}>Increment</button>
      <span>{count}</span>
      <button onClick={() => dispatch(decrement())}>Decrement</button>
    </div>
  );
}

// -----------------------------

import { Provider } from "react-redux";
import { store } from "./app/store";
import { Counter } from "./features/counter/Counter";

const App = () => (
  <Provider store={store}>
    <div>
      <Counter />
      <Counter />
    </div>
  </Provider>
);

export default App;
```

쥬스탠드는,

```tsx
type State = {
  counter: {
    value: number;
  };
  counterActions: {
    increment: () => void;
    decrement: () => void;
    incrementByAmount: (amount: number) => void;
  };
};

export const useStore = create<State>((set) => ({
  counter: { value: 0 },
  counterActions: {
    increment: () =>
      set((state) => ({
        counter: { value: state.counter.value + 1 },
      })),
    decrement: () =>
      set((state) => ({
        counter: { value: state.counter.value - 1 },
      })),
    incrementByAmount: (amount: number) =>
      set((state) => ({
        counter: { value: state.counter.value + amount },
      })),
  },
}));

// -----------------------------

import { useStore } from "./store";

export function Counter() {
  const count = useStore((state) => state.counter.value);
  const { increment, decrement } = useStore((state) => state.counterActions);

  return (
    <div>
      <button onClick={increment}>Increment</button>
      <span>{count}</span>
      <button onClick={decrement}>Decrement</button>
    </div>
  );
}

// -----------------------------

import { Counter } from "./features/counter/Counter";

const App = () => (
  <div>
    <Counter />
    <Counter />
  </div>
);

export default App;
```

차이점을 다시 짚자면,

1. 두 라이브러리의 가장 큰 차이점 중 하나는 디렉터리 구조다. Redux는 대규모 어플리케이션에 유용한 패턴인 features 디렉터리 구조를 제안하고 있다. 반면, Zustand는 구조에 대한 의견을 제시하지 않기 때문에 어떻게 파일과 디렉터리를 구성할 것인지는 개발자의 몫으로 남겨둔다.
2. Redux는 `state.value += 1;`과 같은 변경을 허용하기 위해 Immer를 사용한다. -> 🧐 뭔가, 상태가 불변 객체라는 점에서 +=나 -=같은 연산자를 잘 안썼던 거 같아서 어색하다...
3. (?) 책이 앞에서 말한 것과는 조금 다르게 Zustand는 사실상 데이터 흐름 측면에 대한 의견을 제시하지 않는 반면, Redux는 철저히 단방향 데이터 흐름을 기반으로 한다고 말한다.

### Jotai와 Recoil의 비교

차이점을 짚으면,

1. 가장 큰 차이점은 key 문자열의 존재다. Jotai로 개발을 하는 큰 동기 중 하나가 key 문자열을 생략하는 것이다. map과 같은 고차함수를 사용하면서 겪는 어려운 작업 중 하나가 고유한 key의 이름인데, 이러한 작업을 건너뛸 수 있다.
2. 구현 측면에서 Jotai는 WeakMap을 활용하고 아톰 객체의 참조에 의존한다. 반면, Recoil은 객체 참조에 의존하지 않는 key 문자열을 기반으로 한다.
3. Jotai의 atom 함수는 Recoil의 atom과 selector 두 가지 모두를 대체한다.
4. Jotai의 공급자 제거 모드는 Provider 컴포넌트를 생략할 수 있게 해준다.

### Valtio와 MobX 비교

먼저 유사한 측면은, 둘 다 변경 가능한 상태를 기반으로 하여 사용법이 비슷하다.

차이점은,

1. MobX는 클래스 기반인 반면에 Valtio는 객체 기반이다. 보통 Valtio는 특정 스타일을 강요하지 않는다.
2. 렌더링 최적화 방식에서 차이가 있다. MobX는 옵저버 방식을 택하는 반면, Valtio는 훅 방식을 택한다. 옵저버 방식은 더 예측 가능성이 높고 훅 방식은 동시성 렌더링에 더 친화적이다.
