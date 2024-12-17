# 08. 사용 사례 시나리오 2: Jotai

> bundle size : 3.5 KiB | downloads : 64M

- Jotai - 일본어로 **"상태(状態)"**
- 아톰 모델
- 컴포넌트 상태 사용, 불변 상태 모델
- 컨텍스트 + 구독 패턴 사용
- 아톰 자체는 값을 가지지 않기에 모듈 상태와 달리 한번 정의한 아톰 재사용 가능
- 배율 구조로 리렌더링을 최적화하는 `Atoms-in-Atom`이라는 패턴이 있다.
- [뭔가 상당히 많음](https://jotai.org/docs)

## 사용하기

기본적인 Context API vs Jotai 비교하기

- 아래는 기본적인 카운터 증가 컴포넌트이다.
- Context API와 Jotai의 사용 방법은 크게 다르지 않아 보인다.
- 그렇다면 이런 형태를 만들기 위해 필요한 제반 작업은 무엇일까?

```tsx
const Counter2 = () => {
  // =============================================
  // --> use Context API
  const [count, setCount] = useContext(CountContext);
  // --> use Jotai
  const [count, setCount] = useAtom(countAtom);
  // =============================================
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};
```

- Context API의 경우

```tsx
// Context 생성
const CountContext = createContext(
  undefined as unknown as [number, Dispatch<SetStateAction<number>>]
);
// Provider 생성
const CountProvider = ({ children }: { children: ReactNode }) => (
  <CountContext.Provider value={useState(0)}>{children}</CountContext.Provider>
);
// Context 감싸기
const App = () => <CountProvider>{/* // ... 생략 */}</CountProvider>;
// 컴포넌트에서 사용하기
const [count, setCount] = useContext(CountContext);
```

- Jotai의 경우

```tsx
// 필요한 atom 생성
const countAtom = atom(0);
// 컴포넌트에서 사용하기
const [count, setCount] = useAtom(countAtom);
```

- Jotai에서 정의 하는 atom 이란? 주저리 주저리… GPT야 번역해줘!
  - The `atom` function is to create an atom config
    - 아톰 는 아톰 구성을 생성하는 것입니다.
  - We call it "atom config" as it's just a definition and it doesn't yet hold a value.
    - 이 함수는 정의일 뿐이며 아직 값을 보유하지 않기 때문에 `atom config`라고 부릅니다.
  - We may also call it just "atom" if the context is clear.
    - 문맥이 명확하다면 그냥 `atom`이라고 부를 수도 있습니다.
  - An atom config is an immutable object.
    - 아톰 구성은 불변 객체입니다.
  - The atom config object doesn't hold a value. The atom value exists in a store.
    - 아톰 구성 개체는 값을 보유하지 않습니다. 아톰 값은 스토어에 존재합니다.
  - To create a primitive atom (config), all you need is to provide an initial value.
    - 원시 아톰(config)을 만들려면 초기 값만 제공하면 됩니다.

```tsx
function atom<Value, Args extends unknown[], Result>(
  read?: Value | Read<Value, SetAtom<Args, Result>>,
  write?: Write<Args, Result>
) {
  const key = `atom${++keyCount}`;
  const config = {
    toString() {
      return import.meta.env?.MODE !== "production" && this.debugLabel
        ? key + ":" + this.debugLabel
        : key;
    },
  } as WritableAtom<Value, Args, Result> & { init?: Value | undefined };
  if (typeof read === "function") {
    config.read = read as Read<Value, SetAtom<Args, Result>>;
  } else {
    config.init = read;
    config.read = defaultRead;
    config.write = defaultWrite as unknown as Write<Args, Result>;
  }
  if (write) {
    config.write = write;
  }
  return config;
}
```

- useAtomValue : read-only atom
- useSetAtom : updater
  - `const [, setValue] = useAtom(valueAtom)` 형태처럼 value가 필요없을 때 유용

## 활용하기

```tsx
// primitive atom
function atom<Value>(initialValue: Value): PrimitiveAtom<Value>;
// use ==========================================
const countAtom = atom(0);
```

- `initialValue`: 값이 변경될 때까지 아톰이 반환할 초기 값.

```tsx
// read-only atom
function atom<Value>(read: (get: Getter) => Value): Atom<Value>
// use ==========================================
const countAtom = atom(1)
const readOnlyAtom = atom((get) => get(countAtom))
// read-only derived atom도 가능
const doubleReadOnlyAtom = atom((get) => get(countAtom) * 2)

// in component
const [value, setValue] = useAtom(readOnlyAtom);

// write 시도 하면?
<button
  onClick={() => {
    setValue(4000);
  }}
>
  click
</button>

// 에러 반환
Error
not writable atom
```

- `read` : 아톰을 읽을 때마다 평가되는 함수.
- `get`은 `atom config`을 받아 `Provider`에 저장된 값을 반환하는 함수입니다.
- 종속성이 추적되므로, 한 아톰에 대해 `get`이 한 번 이상 사용되면 아톰 값이 변경될 때마다 읽기가 다시 평가됩니다

```tsx
// writable derived atom
function atom<Value, Args extends unknown[], Result>(
  read: (get: Getter) => Value,
  write: (get: Getter, set: Setter, ...args: Args) => Result
): WritableAtom<Value, Args, Result>;

// use ==========================================
const priceAtom = atom(1000);
const readWriteAtom = atom<number, number[], void>(
  (get) => get(priceAtom) * 2,
  (get, set, newPrice, new2Price) => {
    set(priceAtom, newPrice / 4);
  }
);

<button
  onClick={() => {
    setValue(10000, 40000);
  }}
>
  click
</button>;

// 1. initial -> 1000
// 2. click!
// 3. log -> 10000 / updated -> 5000
// 3-1. 10000 -> 10000 * 2 -> 20000 / 4 -> 5000
```

- `write` : 아톰의 값을 변경할 때 주로 사용되는 함수
- 더 나은 설명을 위해 반환된 `useAtom`에서 반환되는 튜플의 1번 인덱스인 `useAtom()[1]`을 호출할 때마다 호출 된다,
- 원시 아톰에서 이 함수의 기본값은 해당 아톰의 값을 변경한다
- `get`은 위에서 설명한 것과 유사하지만 종속성을 추적하지 않는다.
  - writable을 위해서 참조만 하는 듯? 합니다. `get` 호출하면 `prev` 값을 반환한다.
- `set`은 아톰 구성과 새 값을 받아 Provider의 아톰 값을 업데이트하는 함수이다
- `...args는 useAtom()[1]`을 호출할 때 받는 인수이다.
- 결과는 write 함수의 반환값이다.

```tsx
// write-only derived atom
function atom<Value, Args extends unknown[], Result>(
  read: Value,
  write: (get: Getter, set: Setter, ...args: Args) => Result
): WritableAtom<Value, Args, Result>;

// use ==========================================
const priceAtom = atom(1000);
const readWriteAtom = atom<null, number[], void>(null, (get, set, newPrice) => {
  set(priceAtom, newPrice / 4);
});

const Test = () => {
  const price = useAtomValue(priceAtom);
  const setValue = useSetAtom(readWriteAtom);
  return (
    <div>
      {price}
      <button
        onClick={() => {
          setValue(10000);
        }}
      >
        click
      </button>
    </div>
  );
};
```

- `write-only` 일 때는 `useSetAtom` 이 아닌 `useAtom` 을 사용하더라도 값을 반환하는 튜플의 0번 인덱스를 가져와도 `null` 을 파라미터로 넣어서 `value`만 반환된다.
- 일종의 트릭인듯?

## 최적화

- 책의 내용 처럼 불필요한 렌더링 방지를 위해 사용되지 않는 atom을 제외하는 작업을 하는게 뭔가… 뭔가함..
- 파생되는 무언가를 만들 수는 없을까?

## Atoms-in-Atom

- 말그대로 atom안에서 atom들을 관리하는 방식
- UID를 활용한 렌더링 관리
