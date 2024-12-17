# 07. 사용 사례 시나리오 1: Zustand

> bundle size : 588 B | downloads : 238M

zustand는 사용해본 라이브러리라 나름의 정리를 해보았다.

zustand가 독일어로 “상태”라는 뜻이다

책은 v3로 되어 있어서 v5 기준으로 작성해보겠다

## 모듈 상태와 불변 상태 이해하기

- 모듈 상태를 위해 설계된 store
- 불변 상태 모델 기반 -> 새 객체를 반환하여 업데이트
- 수정하지 않은 객체를 재사용
- 불변 상태의 장점
  - 참조에 대한 동등성 확인만 하면 변경 유무를 알 수 있다. -> 매번 새로 만들어서 업데이트 하기 때문
  - 즉, 객체의 값 전체를 확인할 필요가 없다는 뜻

## 왜 더 나음?

### Why zustand over redux?

- Simple and un-opinionated
- Makes hooks the primary means of consuming state
- Doesn't wrap your app in context providers
- [Can inform components transiently (without causing render)](https://github.com/pmndrs/zustand?tab=readme-ov-file#transient-updates-for-often-occurring-state-changes)

  ```tsx
  const useScratchStore = create((set) => ({ scratches: 0, ... }))

  const Component = () => {
    // Fetch initial state
    const scratchRef = useRef(useScratchStore.getState().scratches)
    // Connect to the store on mount, disconnect on unmount, catch state-changes in a reference
    useEffect(() => useScratchStore.subscribe(
      state => (scratchRef.current = state.scratches)
    ), [])
    ...
  ```

### Why zustand over context?

- Less boilerplate
- Renders components only on changes
- Centralized, action-based state management

## 간단한 사용 방법

- 이는 컴포넌트 외부에서 상태를 읽고 쓰는 방법
- 리액트 서버 컴포넌트 or next 13 버전 이상에서는 추천하지 않는다고 한다.
- 서버에는 상태 라는 개념이 없기? 때문 [#2200](https://github.com/pmndrs/zustand/discussions/2200).

```jsx
// store 생성
const store = create(() => ({ count: 0 }));

// 값 가져오기
store.getState(); // { count: 0 }

// 올바른 업데이트
store.setState({ count: 1 });
// 잘못된 업데이트
const state1 = store.getState(); // { count: 1 }
state1.count = 2;
store.setState(state1);
```

### store 생성

- [바닐라의 경우 이전에 배웠던 구독 모델](https://github.com/pmndrs/zustand/blob/main/src/vanilla.ts#L58)
- [리액트의 경우 바닐라의 구독을 기반으로 `useSyncExternalStore` 훅을 사용](https://github.com/pmndrs/zustand/blob/main/src/react.ts)

```jsx
const store = create(() => ({ count: 0, text: "" }));
```

### 상태를 병합하고 업데이트 하는 방식

보통은 스프레드 연산자를 쓰는데 zustand에서는 생략 가능하다

- setState

```jsx
const store = create(() => ({ count: 0 , text: ''}));
store.setState({ count: 1 }) // 스프레드 생략
console.log(store.getState()) // { count: 1 , text: ''}

// ======= how? =======

const setState: StoreApi<TState>['setState'] = (partial, replace) => {
  // TODO: Remove type assertion once https://github.com/microsoft/TypeScript/issues/37663 is resolved
  // https://github.com/microsoft/TypeScript/issues/37663#issuecomment-759728342
  const nextState =
    typeof partial === 'function'
      ? (partial as (state: TState) => TState)(state)
      : partial
  if (!Object.is(nextState, state)) { //Object.is 얕은 비교
    const previousState = state
    state =
      (replace ?? (typeof nextState !== 'object' || nextState === null))
        ? (nextState as TState)
        : Object.assign({}, state, nextState) // Object.assign
    listeners.forEach((listener) => listener(state, previousState)) // 구독
  }
}
```

- store 내부의 action에서 업데이트 하는 방식
  - 중첩 객체에서는 스프레드 연산자를 써야 한다.
  - 중첩이 심해진다면 [라이브러리](https://github.com/pmndrs/zustand/blob/main/docs/guides/updating-state.md#deeply-nested-object)의 도움을 받는다.
  - state는 불변이기 때문에 상태 업데이트를 위해 `set` 메소드를 사용한다.

```jsx
const useCountStore = create((set) => ({
    count: 0,
    nested: { count: 0 },
    inc: () => set((state) => ({ count: state.count + 1 })),
}))

// 스프레드 사용
set((state) => ({ ...state, count: state.count + 1 }))
// 스프레드 생략
set((state) => ({ count: state.count + 1 }))
// 중첩 객체
set((state) => ({
  nested: { ...state.nested, count: state.nested.count + 1 },
})),
// get을 이용한 방식
const useCountStore = create((set, get) => ({}));
set((state) => ({ count: get().count + 1 }))
```

## 리액트 훅을 이용한 리렌더링 최적화

- 수동 최적화 → 선택자 함수 사용

```jsx
const useBearStore = create((set) => ({
  nuts: 0,
  honey: 0,
  treats: {},
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
}));

// 최적화 전
const { nuts, honey } = useBearStore();

// 최적화 후
const nuts = useBearStore((state) => state.nuts);
const honey = useBearStore((state) => state.honey);
```

- `useShallow`를 써서도 가능

```jsx
// object에서 nust or honey 변경 시 -> 리렌더링
const { nuts, honey } = useBearStore(
  useShallow((state) => ({ nuts: state.nuts, honey: state.honey }))
);

// array에서 nust or honey 변경 시 -> 리렌더링
const [nuts, honey] = useBearStore(
  useShallow((state) => [state.nuts, state.honey])
);

// Mapped형태에서 treats의 순서, 개수, 키값 변경시 -> 리렌더링
const treats = useBearStore(useShallow((state) => Object.keys(state.treats)));
```

- 들어가기 전에 store 생성 방식의 차이가 있다.

```tsx
import { create } from "zustand";

interface BearState {
  bears: number;
  increase: (by: number) => void;
}

// 책에서
const useBearStore = create<BearState>((set) => ({}));

// 책이랑 다름 / 커링씀
const useBearStore = create<BearState>()((set) => ({
  bears: 0,
  increase: (by) => set((state) => ({ bears: state.bears + by })),
}));
```

- [왜 커링을 쓸까?](https://github.com/pmndrs/zustand/blob/main/docs/guides/typescript.md) 타스는 어려워!

## 사용 방법 중 하나 : sliced-pattern

- [sliced-pattern](https://github.com/pmndrs/zustand/blob/main/docs/guides/slices-pattern.md) 을 활용한 설계
- 종종 써먹던 패턴
- 메인 스토어가 커질 때 관심사에 따라 한번 더 모듈 단위로 쪼개서 관리하는 방법

- 서브 스토어들

```tsx
export const createFishSlice = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
});

export const createBearSlice = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
});
```

- 메인 스토어

```tsx
import { create } from "zustand";
import { createBearSlice } from "./bearSlice";
import { createFishSlice } from "./fishSlice";

export const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}));
```

- 리액트에서 사용하기

```tsx
import { useBoundStore } from "./stores/useBoundStore";

function App() {
  const bears = useBoundStore((state) => state.bears);
  const fishes = useBoundStore((state) => state.fishes);
  const addBear = useBoundStore((state) => state.addBear);
  return (
    <div>
      <h2>Number of bears: {bears}</h2>
      <h2>Number of fishes: {fishes}</h2>
      <button onClick={() => addBear()}>Add a bear</button>
    </div>
  );
}

export default App;
```

- 다른 서브 스토어의 값 가져오기

```tsx
export const createBearFishSlice = (set, get) => ({
  addBearAndFish: () => {
    get().addBear();
    get().addFish();
  },
});
```
