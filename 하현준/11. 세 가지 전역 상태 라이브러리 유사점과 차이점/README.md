## Zustand와 Redux의 차이점

Redux를 활용한 상태관리 방식

```tsx
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

const initialState = {
  value: 0,
};

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

// 리듀서와 액션을 각각 추출하여 사용이 가능하다.
export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

```tsx
import { useSelector, useDispatch } from "react-redux";
import { decrement, increment } from "./counterSlice";

export function Counter() {
  // selector를 통해 값을 가져온다.
  const count = useSelector(
    (state: { counter: { value: number } }) => state.counter.value
  );
  // dispatch를 통해 액션을 실행한다.
  const dispatch = useDispatch();
  return (
    <div>
      <button onClick={() => dispatch(increment())}>Increment</button>
      <span>{count}</span>
      <button onClick={() => dispatch(decrement())}>Decrement</button>
    </div>
  );
}
```

Zustand를 활용한 상태관리 방식

```tsx
import create from "zustand";

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
      set((state) => ({ counter: { value: state.counter.value + 1 } })),
    decrement: () =>
      set((state) => ({ counter: { value: state.counter.value - 1 } })),
    incrementByAmount: (amount: number) =>
      set((state) => ({ counter: { value: state.counter.value + amount } })),
  },
}));
```

```tsx
import { useStore } from "./store";

export function Counter() {
  const count = useStore((state) => state.counter.value);
  const { increment, decrement } = useStore((state) => state.counterActions);
  return (
    <div>
      <div>
        <button onClick={increment}>Increment</button>
        <span>{count}</span>
        <button onClick={decrement}>Decrement</button>
      </div>
    </div>
  );
}
```

차이점

- 서로 다른 디렉토리 구조
  - Redux: features 디렉토리 구조
  - Zustand: 개발자의 몫
- Redux는 스토어 생성에 immer를 사용, Zustand는 immer를 사용하지 않음
- Redux는 컨텍스트를 사용하여 상태를 전파, Zustand는 import를 통해 상태를 관리,
  - ❓Zustand도 선택적으로 컨텍스트를 제공한다는데 같이 사용할일이 있으려나

요약: Redux는 상태관리를 위한 권장되는 패턴을 제시해주지만, Zustand는 의견을 따로 제시하지 않고 개발자에게 맡긴다.

## Recoil과 Jotai의 비교

- key 문자열의 존재. Jotai는 key문자열을 사용하지 않는다. Recoil은 key 문자열을 기반으로 하기 때문에 직렬화가 가능하다.
- Jotai의 atom은 Recoil의 selector + atom을 모두 포함
- Jotai의 공급자 제거 모드를 통해 Provider 컴포넌트 생략이 가능하게 해준다.

## MobX 와 Valtio의 비교

Mobx의 예시)

```tsx
import { makeAutoObservable } from "mobx";
import { observer } from "mobx-react";

class Timer {
  secondsPassed = 0;
  constructor() {
    makeAutoObservable(this);
  }
  increase() {
    this.secondsPassed += 1;
  }
  reset() {
    this.secondsPassed = 0;
  }
}

const myTimer = new Timer();

setInterval(() => {
  myTimer.increase();
}, 1000);

const TimerView = observer(({ timer }: { timer: Timer }) => (
  <button onClick={() => timer.reset()}>
    Seconds passed: {timer.secondsPassed}
  </button>
));

const App = () => (
  <>
    <TimerView timer={myTimer} />
  </>
);

export default App;
```

Valtio의 예시)

```tsx
import { proxy, useSnapshot } from "valtio";

const myTimer = proxy({
  secondsPassed: 0,
  increase: () => {
    myTimer.secondsPassed += 1;
  },
  reset: () => {
    myTimer.secondsPassed = 0;
  },
});

setInterval(() => {
  myTimer.increase();
}, 1000);

const TimerView = ({ timer }: { timer: typeof myTimer }) => {
  const snap = useSnapshot(timer);
  return (
    <button onClick={() => timer.reset()}>
      Seconds passed: {snap.secondsPassed}
    </button>
  );
};

const App = () => (
  <>
    <TimerView timer={myTimer} />
  </>
);

export default App;
```

차이점

- MobX는 클래스 기반, Valtio는 객체 기반
- Valtio의 상태 객체에서 함수를 분리해낼 수 있음
  ```tsx
  const myTimer = proxy({
    secondsPassed: 0,
  });

  const increase = () => {
    timer.secondsPassed += 1;
  };

  const reset = () => {
    timer.secondsPassed = 0;
  };
  ```
  즉, 번들 크기가 최적화될 것이다.
- 렌더링 최적화 방식이 서로 다르다. MobX는 옵저버 방식, Valtio는 훅 방식을 사용

## Zustand, Jotai, Valtio

세 라이브러리의 차이점

**생태가 어디에 위치하는가?**

- 모듈 상태: 모듈 수준에서 생성되는 상태이고, 리액트에 속하지 않는 상태 (Zustand, Valtio)
- 컴포넌트 상태: 리액트 컴포넌트 생명주기에서 생성되고 리액트에 의해 제어되는 상태 (Jotai)

**상태 갱신 스타일은 무엇인가?**

- Zustand는 불변 상태 모델을 기반
  - 장점: 객체 참조를 비교해서 변경 사항이 있는지 파악이 가능
- Valtio는 변경 가능한 상태 모델을 기반
  - 장점: 객체가 깊이 중첩되어 있는 경우에 편함
