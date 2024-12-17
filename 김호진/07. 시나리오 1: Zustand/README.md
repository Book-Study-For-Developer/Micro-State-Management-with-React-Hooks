## 시나리오 1: Zustand

Zustand는 주로 리액트 모듈 상태를 생성하도록 설계된 작은 라이브러리이다. 상태 객체를 수정할 수 없고 항상 새로 만들어야 하는 불변 갱신 모델을 기반으로 한다.

### 모듈 상태와 불변 상태 이해하기

Zustand는 상태를 변경하기 위해서는 새 객체를 생성해서 대체해야 하며, 수정하지 않은 객체는 재사용해야 한다. 불변 상태의 장점은 상태 객체의 참조에 대한 동등성만 확인하면 변경 여부를 알 수 있다.

```jsx
const store = create(() => ({count : 0}));

console.log(store.getState()); // {count : 0}
store.setState({count : 1});
console.log(store.getState()); // {count : 1}

```

여러개의 속성을 가질때의 예제를 아래와 같이 살펴보자

```jsx
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

```jsx
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

```jsx
const Component = () => {
  const count = useStore((state) => state.count);
  return <div>count : {count}</div>;
}

```

위와 같은 선택자 기반 리렌더링 제어를 **수동 렌더링 최적화**라고 한다.

### Zustand Middleware

Zustand를 간단하게는 사용해본적이 있는데 middleware에 대해서는 살펴본적이 없어서 해당 부분을 가볍게 살펴봤다. 

**Combine을 사용할때**

```jsx
const initialState = {
  todos: [] as string[],
  loading: false
}

const useTodoStore = create(
  combine(
    initialState,
    (set) => ({
      addTodo: (todo: string) => set((state) => ({ 
        todos: [...state.todos, todo] 
      })),
      toggleLoading: () => set((state) => ({ 
        loading: !state.loading 
      }))
    })
  )
)
```

**Combine을 사용하지 않을때**

```jsx
interface TodoState {
  todos: string[]
  loading: boolean
  addTodo: (todo: string) => void
  toggleLoading: () => void
}

const useTodoStore = create<TodoState>((set) => ({
  todos: [],
  loading: false,
  addTodo: (todo) => set((state) => ({ 
    todos: [...state.todos, todo] 
  })),
  toggleLoading: () => set((state) => ({ 
    loading: !state.loading 
  }))
}))
```

Combine을 사용할때와 사용하지 않을때의 차이점을 살펴봤는데 Combine을 사용했을때 초기 상태와 액션이 명확하게 분리되고 타입 추론도 비교적 잘 이루어지는 것 같아서 복잡한 상태를 다루거나 할때 용이하게 사용할 수 있을 것 같다.

Combine 실제 내부 구현체도 살펴봤는데 비교적 간단하게 구성되어 있었다. 

```jsx
import type { StateCreator, StoreMutatorIdentifier } from '../vanilla.ts'

type Write<T, U> = Omit<T, keyof U> & U

type Combine = <
  T extends object,
  U extends object,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = [],
>(
  initialState: T,
  additionalStateCreator: StateCreator<T, Mps, Mcs, U>,
) => StateCreator<Write<T, U>, Mps, Mcs>

export const combine: Combine =
  (initialState, create) =>
  (...a) =>
    Object.assign({}, initialState, (create as any)(...a))
```

타입을 제외하고 살펴보면 Curring을 사용하고 빈 객체와 initialState를  create를 통해 만들어진 함수의 결과와 병합하는식으로 동작한다.

**Immer**

Immer의 경우에는 대부분 아실 것 같은데 불변성을 지키기 위해서 사용하는 라이브러리인 Immer.js와 동일한 역할을 한다. 

```jsx
const useStore = create((set) => ({
  user: {
    profile: {
      personal: {
        name: 'John',
        address: {
          city: 'Seoul',
          street: 'Main St'
        }
      }
    }
  },
  updateStreet: (newStreet) => set((state) => ({
    user: {
      ...state.user,
      profile: {
        ...state.user.profile,
        personal: {
          ...state.user.profile.personal,
          address: {
            ...state.user.profile.personal.address,
            street: newStreet
          }
        }
      }
    }
  }))
}))
```

위와 같이 spread 연산자를 활용해서 안쪽 깊이 있는 객체의 특정 요소를 변경하려고 할때 아래와 같이 간단하게 사용할 수 있다. 

```jsx
const useStore = create(
  immer((set) => ({
    user: {
      profile: {
        personal: {
          name: 'John',
          address: {
            city: 'Seoul',
            street: 'Main St'
          }
        }
      }
    },
    updateStreet: (newStreet) => set((state) => {
      state.user.profile.personal.address.street = newStreet
    })
  }))
)
```

zustand Immer 내부 구조도 간단하게 살펴봤는데 역시나 immer.js 라이브러리의 produce 함수를 사용하고 있다. 

```jsx
import { produce } from 'immer'
import type { Draft } from 'immer'
import type { StateCreator, StoreMutatorIdentifier } from '../vanilla.ts'

type Immer = <
  T,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = [],
>(
  initializer: StateCreator<T, [...Mps, ['zustand/immer', never]], Mcs>,
) => StateCreator<T, Mps, [['zustand/immer', never], ...Mcs]>

declare module '../vanilla' {
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  interface StoreMutators<S, A> {
    ['zustand/immer']: WithImmer<S>
  }
}

type Write<T, U> = Omit<T, keyof U> & U
type SkipTwo<T> = T extends { length: 0 }
  ? []
  : T extends { length: 1 }
    ? []
    : T extends { length: 0 | 1 }
      ? []
      : T extends [unknown, unknown, ...infer A]
        ? A
        : T extends [unknown, unknown?, ...infer A]
          ? A
          : T extends [unknown?, unknown?, ...infer A]
            ? A
            : never

type SetStateType<T extends unknown[]> = Exclude<T[0], (...args: any[]) => any>

type WithImmer<S> = Write<S, StoreImmer<S>>

type StoreImmer<S> = S extends {
  setState: infer SetState
}
  ? SetState extends {
      (...a: infer A1): infer Sr1
      (...a: infer A2): infer Sr2
    }
    ? {
        // Ideally, we would want to infer the `nextStateOrUpdater` `T` type from the
        // `A1` type, but this is infeasible since it is an intersection with
        // a partial type.
        setState(
          nextStateOrUpdater:
            | SetStateType<A2>
            | Partial<SetStateType<A2>>
            | ((state: Draft<SetStateType<A2>>) => void),
          shouldReplace?: false,
          ...a: SkipTwo<A1>
        ): Sr1
        setState(
          nextStateOrUpdater:
            | SetStateType<A2>
            | ((state: Draft<SetStateType<A2>>) => void),
          shouldReplace: true,
          ...a: SkipTwo<A2>
        ): Sr2
      }
    : never
  : never

type ImmerImpl = <T>(
  storeInitializer: StateCreator<T, [], []>,
) => StateCreator<T, [], []>

const immerImpl: ImmerImpl = (initializer) => (set, get, store) => {
  type T = ReturnType<typeof initializer>

  store.setState = (updater, replace, ...a) => {
    const nextState = (
      typeof updater === 'function' ? produce(updater as any) : updater
    ) as ((s: T) => T) | T | Partial<T>

    return set(nextState, replace as any, ...a)
  }

  return initializer(store.setState, get, store)
}

export const immer = immerImpl as unknown as Immer
```

해당 부분 역시 타입을 제외하고 구현체만 살펴보면 curring 구조로 되어 있고 store에 setState 함수를 재정의하여 function 타입일 경우에는 immer의 produce 함수를 통해 불변성을 보장한다.