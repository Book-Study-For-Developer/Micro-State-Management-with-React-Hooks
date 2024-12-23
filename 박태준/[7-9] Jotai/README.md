### Jotai 모델과 리렌더링 최적화

jotai는 컴포넌트 상태를 사용하며, 불변 상태 모델을 따른다. 내부적으로 Context API를 사용하여 구현되어 있다.

**구문 단순성**

jotai에서는 Context API와 달리 atom마다 Provider를 따로 생성할 필요가 없다. atom을 만들고 useAtom으로 사용하기만 하면 된다.

**파생 아톰(Derived atoms)**

atom들은 가능한 조각(원시값이면 좋다)으로 만들고 이미 존재하는 atom들을 구성하여 파생 아톰을 만들수 있다.

최종적으로 atom이 반환하는 값이 바뀔때만 리렌더링이 일어난다.

작은 atom을 만들고 이를 결합해 더 큰 아톰을 만드는 방식으로 이를 상향식(bottom-up) 접근법이라고 한다.

```tsx
const progressAtom = atom((get) => {
  const anime = get(animeAtom);
  return anime.filter((item) => item.watched).length / anime.length; // atom이 반환하는 값이 바뀔때만 리렌더링이 일어난다.
});
```

특히 객체형 상태를 atom들의 조합으로 구성할 경우 따로 렌더링 최적화를 할 필요가 없다.

```tsx
const firstNameAtom = atom(""); // firstName이 필요하면 구독
const lastNameAtom = atom(""); // lastName이 필요하면 구독
const ageAtom = atom(0);

const personAtom = atom((get) => ({
  firstName: get(firstNameAtom),
  lastName: get(lastNameAtom),
  age: get(ageAtom),
}));

const fullNameAtom = atom((get) => ({
  // 파생 아톰 ageAtom 바뀌어도 리렌더링 되지 않음
  firstName: get(firstNameAtom),
  lastName: get(lastNameAtom),
}));
```

primitve atom 을 기반으로 Read-only atom, Write-only atom, Read-Write atom 3가지 타입의 derived atom을 만들어서 사용할 수 있다. https://www.jotai.com.cn/docs/core/atom

```tsx
import { atom } from "jotai";
const priceAtom = atom(10);
const messageAtom = atom("hello");
const productAtom = atom({ id: 12, name: "good stuff" });

const readOnlyAtom = atom((get) => get(priceAtom) * 2);
const writeOnlyAtom = atom(
  null, // it's a convention to pass `null` for the first argument
  (get, set, update) => {
    // `update` is any single value we receive for updating this atom
    set(priceAtom, get(priceAtom) - update.discount);
    // or we can pass a function as the second parameter
    // the function will be invoked,
    //  receiving the atom's current value as its first parameter
    set(priceAtom, (price) => price - update.discount);
  }
);
const readWriteAtom = atom(
  (get) => get(priceAtom) * 2,
  (get, set, newPrice) => {
    set(priceAtom, newPrice / 2);
    // you can set as many atoms as you want at the same time
  }
);
```

### Jotai가 아톰 값을 저장하는 방법

atom 자체는 값을 가지지 않는다. 아래와 같이 아톰 모듈을 생성했다고 해서 아톰 상태가 해당 모듈에서 관리 되지는 않는다. (이부분을 모르고 그동안 사용하고 있었다.)

```tsx
// atom.ts
const countAtom = atom(0);
```

아톰 구성은 값을 가지지 않고 정의일 뿐이다. 상태는 모듈 수준에서 정의된 store에서 관리된다.

이는 키가 아톰 구성 객체이고 값이 아톰 값인 WeakMap이다.

따라서 Provider를 사용해 하위트리에서 별개의 생명주기로 atom을 재활용 할 수 있다.

### Atoms-in-Atom

https://jotai.org/docs/guides/atoms-in-atom

`atom()` 은 상태를 만드는 것이 아니라 아톰 정의(atom config)를 생성한다.

`anAtom.toString()` 은 UUID를 반환하여 key로써 활용할 수 있다.

배열 atom 관리시 배열에 새로운 값을 추가할때 atom을 만들어서 추가하는 패턴 등에 적용될 수 있다.

```tsx
const countsAtom = atom([atom(1), atom(2), atom(3)]); // 배열 아톰

const Counter = ({ countAtom }) => {
  const [count, setCount] = useAtom(countAtom);
  return (
    <div>
      {count} <button onClick={() => setCount((c) => c + 1)}>+1</button>
    </div>
  );
};

const Parent = () => {
  const [counts, setCounts] = useAtom(countsAtom);
  const addNewCount = () => {
    const newAtom = atom(0); // 배열에 새로운 아톰을 만들어서 추가한다.
    setCounts((prev) => [...prev, newAtom]);
  };
  return (
    <div>
      {counts.map((countAtom) => (
        <Counter countAtom={countAtom} key={countAtom} /> // key로 아톰 자체를 사용한다. UUID이다.
      ))}
      <button onClick={addNewCount}>Add</button>
    </div>
  );
};
```

### action atom

상태를 변경하기 위해 함수를 만드는 것처럼 atom()의 두번째 인수인 write 함수만 사용하여 액션함수를 만들수 있다. 실제로 회사에서도 활용하고 있다.

```tsx
const countAtom = count(0);
const incrementCountAtom = atom(
  null, // 첫번째 인수로는 null을 넣는다.
  (get, set, arg) => set(countAtom, (c) => c + 1)
);
```

```tsx
import { useSetAtom } from "jotai";

const IncrementButton = () => {
  const increment = useSetAtom(incrementCountAtom);
  return <button onClick={increment}>inc</button>;
};
```

### atomWithStorage

말 그대로 로컬 스토리지, 세션스토리지에 값을 저장할 수 있게 하는 atom이다. 최근에 사용한 적이 있어서 추가로 정리.

https://jotai.org/docs/utilities/storage

```tsx
export const modeAtom = atomWithStorage<"none" | "chat">(
  "mode", // key
  "none", // initialVlaue
  undefined, // storage, 세번째 옵션을 지정하지 않으면 로컬 스토리지를 사용한다.
  {
    getOnInit: true,
  }
);
```

리액트에서 사용시에 컴포넌트 마운트시에는 스토리지에서 값을 가져오지 않고 initialValue를 가지고 렌더링 되며 이는 의도된 구현이다. 관련 이슈 (https://github.com/pmndrs/jotai/issues/2240)

`getOnInit: true` 옵션을 추가해야 첫 마운트시에도 값이 동기화된후에 컴포넌트가 실행된다. 이전 사용버전에서는 지원하지 않았어서 최근에 버전업을 했다.

### 라이브러리 살펴보기 (리렌더링, 구독 구현)

jotai에서는 `useAtom`을 사용하면 자동으로 해당 atom을 구독하고 이후 값이 바뀌면 리렌더링이 일어난다.어떻게 자동으로 추적하는지 궁금해서 찾아보았다.

https://github.com/pmndrs/jotai/blob/main/src/react/useAtom.ts 에서는 useAtom이 useAtomValue를 반환하고 있다.

```tsx
export function useAtom<Value, Args extends unknown[], Result>(
  atom: Atom<Value> | WritableAtom<Value, Args, Result>,
  options?: Options
) {
  return [
    useAtomValue(atom, options),
    // We do wrong type assertion here, which results in throwing an error.
    useSetAtom(atom as WritableAtom<Value, Args, Result>, options),
  ];
}
```

useAtomValue 구현부를 보면 리렌더링을 어떻게 유발하는지 확인 할 수 있다. https://github.com/pmndrs/jotai/blob/main/src/react/useAtomValue.ts

use hook 도입을 위해 Promise 관련 구현이 조금 섞여있지만.. useReducer를 사용해서 받은 rerender(dispatch)를 사용해서 리렌더링을 해주고 있는것으로 보인다. 아래 useEffect에 구독을 해주고 있고 콜백으로 rerender를 넘겨준다. 또한 이전장들에서 살펴본 거처럼 첫 구독시에 한번 실행해준다.

```tsx
export function useAtomValue<Value>(atom: Atom<Value>, options?: Options) {
  const store = useStore(options);

  // rerender -> dispatch로 리렌더링을 유발할 수 있다.
  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer<
      ReducerWithoutAction<readonly [Value, Store, typeof atom]>,
      undefined
    >(
      (prev) => {
        const nextValue = store.get(atom);
        if (
          Object.is(prev[0], nextValue) &&
          prev[1] === store &&
          prev[2] === atom
        ) {
          return prev;
        }
        return [nextValue, store, atom];
      },
      undefined,
      () => [store.get(atom), store, atom]
    );

  let value = valueFromReducer;
  if (storeFromReducer !== store || atomFromReducer !== atom) {
    rerender();
    value = store.get(atom);
  }

  const delay = options?.delay;
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      if (typeof delay === "number") {
        const value = store.get(atom);
        if (isPromiseLike(value)) {
          attachPromiseMeta(createContinuablePromise(value));
        }
        // delay rerendering to wait a promise possibly to resolve
        setTimeout(rerender, delay);
        return;
      }
      rerender(); // rerender를 등록
    });
    rerender(); // 처음 구독할 때 한번 실행
    return unsub;
  }, [store, atom, delay]);

  useDebugValue(value);
  // The use of isPromiseLike is to be consistent with `use` type.
  // `instanceof Promise` actually works fine in this case.
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(value);
    return use(promise);
  }
  return value as Awaited<Value>;
}
```

store.sub는 어떻게 구현되어 있는지 타고 올라가 보았다. store는 Provider에서 가져온다.

https://github.com/pmndrs/jotai/blob/main/src/react/Provider.ts

useRef로 store 참조를 저장하고 있는것과(store가 최초 한번만 생성되도록 하기 위함) 확인
내부적으로 Context API를 사용하고 있는거 확인(책 내용 확인)

```tsx
import { createContext, createElement, useContext, useRef } from "react";
import type { FunctionComponentElement, ReactNode } from "react";
import { createStore, getDefaultStore } from "../vanilla.ts";

type Store = ReturnType<typeof createStore>;

type StoreContextType = ReturnType<typeof createContext<Store | undefined>>;
const StoreContext: StoreContextType = createContext<Store | undefined>(
  undefined
);

type Options = {
  store?: Store;
};

export const useStore = (options?: Options): Store => {
  const store = useContext(StoreContext);
  return options?.store || store || getDefaultStore();
};

/* eslint-disable react-compiler/react-compiler */
// TODO should we consider using useState instead of useRef? // 이건 뭘까;;
export const Provider = ({
  children,
  store,
}: {
  children?: ReactNode;
  store?: Store;
}): FunctionComponentElement<{ value: Store | undefined }> => {
  const storeRef = useRef<Store>();
  if (!store && !storeRef.current) {
    storeRef.current = createStore(); // 여기에서 스토어 생성해서 아래에 context로 내려줌
  }
  return createElement(
    StoreContext.Provider,
    {
      value: store || storeRef.current,
    },
    children
  );
};
```

createStore를 찾았고 내부적으로 WeakMap을 사용하고 잇는것을 확인했다. buildStore만 파보면 될거 같다.

https://github.com/pmndrs/jotai/blob/main/src/vanilla/store.ts

```tsx
export const createStore = (): Store => {
  const atomStateMap = new WeakMap();
  const getAtomState = <Value,>(atom: Atom<Value>) => {
    if (import.meta.env?.MODE !== "production" && !atom) {
      throw new Error("Atom is undefined or null");
    }
    let atomState = atomStateMap.get(atom) as AtomState<Value> | undefined;
    if (!atomState) {
      atomState = { d: new Map(), p: new Set(), n: 0 };
      atomStateMap.set(atom, atomState);
    }
    return atomState;
  };
  return buildStore(
    getAtomState,
    (atom, ...params) => atom.read(...params),
    (atom, ...params) => atom.write(...params),
    (atom, ...params) => atom.onMount?.(...params)
  );
};
```

근데 buildStore가 500줄이어서..

https://github.com/pmndrs/jotai/blob/f1861669e81944e3b21667cc5521f50aa0b16f2c/src/vanilla/store.ts#L269C21-L270C1

구독 구현부만 찾아보면 `mounted.l` 에 콜백들을 추가하는것으로 보임

```tsx
type PrdStore = {
  get: <Value>(atom: Atom<Value>) => Value
  set: <Value, Args extends unknown[], Result>(
    atom: WritableAtom<Value, Args, Result>,
    ...args: Args
  ) => Result
  sub: (atom: AnyAtom, listener: () => void) => () => void
  unstable_derive: (fn: (...args: StoreArgs) => StoreArgs) => Store
}

type Store = PrdStore | (PrdStore & DevStoreRev4)

...

const subscribeAtom = (atom: AnyAtom, listener: () => void) => {
    const pending = createPending()
    const atomState = getAtomState(atom)
    const mounted = mountAtom(pending, atom, atomState)
    const listeners = mounted.l
    listeners.add(listener)
    flushPending(pending)
    return () => {
      listeners.delete(listener)
      const pending = createPending()
      unmountAtom(pending, atom, atomState)
      flushPending(pending)
    }
  }

...

  const store: Store = {
    get: readAtom,
    set: writeAtom,
    sub: subscribeAtom,
    unstable_derive,
  }
...

  return store
}

```

마운트 된 atom을 추적하고 이를 통해 atom을 useAtom으로 가져오면 구독이 자동으로 행해지는것 같다.!

```tsx
/**

- State tracked for mounted atoms. An atom is considered "mounted" if it has a
- subscriber, or is a transitive dependency of another atom that has a
- subscriber.
- 
- The mounted state of an atom is freed once it is no longer mounted.
*/
type Mounted = {
  /*Set of listeners to notify when the atom value changes. */
  readonly l: Set<() => void>;
  /*Set of mounted atoms that the atom depends on. */
  readonly d: Set<AnyAtom>;
  /*Set of mounted atoms that depends on the atom. */
  readonly t: Set<AnyAtom>;
  /*Function to run when the atom is unmounted. */
  u?: (pending: Pending) => void;
};
```
