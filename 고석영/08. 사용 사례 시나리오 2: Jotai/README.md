> [!NOTE]
>
> ### Jotai
>
> 전역 상태를 위한 작고 단순한 라이브러리로 리액트의 `useState`, `useReducer`와 함께 상태의 작은 조각인 **아톰** 모델을 기반으로 상태 관리
>
> - 불변 상태 모델: Jotai는 Zustand와 같이 불변 상태 모델을 사용하지만, **컴포넌트 상태**를 사용한다는 점이 차이
> - 의존성 추적: 아톰을 사용하면 Jotai가 자동으로 의존성을 추적 및 의존성에 따라 리렌더링 감지 가능
> - 재사용성: Jotai는 아톰 자체는 값을 가지지 않기 때문에 **모듈 상태와 달리 한번 정의한 아톰을 재사용 가능** → 내부적으로 컨텍스트를 사용하여 아톰을 관리

## Jotai의 특징

### 구문 단순성

```tsx
import { atom, useAtom } from 'jotai'

const countAtom = atom(0)

const Counter1 = () => {
  const [count, setCount] = useAtom(countAtom)
  const inc = () => setCount(c => c + 1)

  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  )
}

const Counter2 = () => {
  const [count, setCount] = useAtom(countAtom)
  const inc = () => setCount(c => c + 1)

  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  )
}

const App = () => (
  <>
    <div>
      <Counter1 />
    </div>
    <div>
      <Counter2 />
    </div>
  </>
)
```

- `useState` 사용하듯이 사용
- 별도의 `Provider` 필요 없음 → 컨텍스트의 ‘기본 스토어’로 내부적으로 자동으로 상태 관리 가능 …하지만 원한다면?
- 선택적 공급자 사용 가능 → 서로 다른 하위 트리에 대해 각각 다른 값을 제공해야 하는 경우
  - 근데 이럴 일이 많이 있을까 싶다…?

### 동적 아톰 생성

- 아톰은 리액트 컴포넌트 생명 주기에 생성되거나 소멸할 수 있음
- Jotai의 `store`는 기본적으로 아톰 구성 객체와 아톰 값으로 구성된 WeakMap 객체
  - key → `atom` 함수로 생성된 아톰 구성 객체(atom config object)
  - value → `useAtom` 훅이 반환하는 값 아톰 값(atom value)

> 즉, `useAtom` 훅이 store에 있는 특정 아톰을 구독한다는 것을 의미하며, 이러한 아톰 기반 구독은 불필요한 리렌더링을 피할 수 있는 기능을 제공한다.

- WeakMap을 사용한 이유?
  - 메모리 관리 → `WeakMap`은 가비지 컬렉션 되지 않아야만 유용한 키에 정보를 매핑할 때 특히 유용한 구조 → 아톰이 더 이상 사용되지 않을 때 GC에 의해 수거하도록 하여 메모리 누수 방지
  - 빠른 조회 및 갱신 → 아톰을 키로 사용해 관련 상태를 저장하고 빠르게 접근 가능
  - (+) WeakMap과 메모리 누수 관련 읽어볼만한 아티클 → [WeakMap이 알고 싶다
    ](https://ui.toast.com/posts/ko_20210901)

## 렌더링 최적화

- store와 선택자 접근 방식 → 모든 것을 저장하는 store와 필요에 따라 store에서 상태를 선택 → **하향식(tod-down) 접근법**
- 아톰 → 작은 아톰을 만들고 이를 결합해 더 큰 아톰 만들기 → **상향식(bottom-up) 접근법**

```tsx
const firstNameAtom = atom('React')
const lastNameAtom = atom('Hooks')
const ageAtom = atom(3)

const PersonComponent = () => {
  const [firstName] = useAtom(firstNameAtom)
  const [lastName] = useAtom(lastNameAtom)
  return (
    <>
      {firstName} {lastName}
    </>
  )
}
```

### 파생 아톰

- 아톰은 원시 타입만큼 작다? → 조작해야 할 아톰이 너무 많을 수 있다는 것을 의미
- 파생 아톰 → 기존 아톰에서 또 다른 아톰을 파생해서 만드는 개념
- atom 함수는 `read` 함수를 받아 파생된 값을 생성할 수 있음
  - `get`: 다른 아톰을 참조하고 그 값을 가져올 수 있는 인수
  - 의존성 자동 추적: 속성 중 하나가 변경될 때마다 리렌더링 → 즉, 셋 중 하나가 변경될 때마다 리렌더링

```tsx
// 이름, 성, 나이를 포함하는 파생 아톰 personAtom
const personAtom = atom((get) => {
	firstName: get(firstNameAtom),
	lastName: get(lastNameAtom),
	age: get(ageAtom)
})
```

- 의존성 추적은 동적이며 조건부 평가에서도 작동
  - `(get) => get(a) ? get(b) : get(c)`
    - a가 참이면 의존성은 a, b
    - b가 참이면 의존성은 a, c

#### 파생 아톰으로 리렌더링 최적화

```tsx
const fullNameAtom = atom((get) => {
	firstName: get(firstNameAtom),
	lastName: get(lastNameAtom)
})

const PersonComponent = () => {
	const person = useAtom(fullNameAtom)

	return <>{person.firstName} {person.lastName}</>
}
```

#### 파생 아톰의 3가지 패턴

```tsx
const priceAtom = atom(10)
```

- 읽기 전용 아톰(Read-only): 다른 아톰의 값을 읽는 아톰 → `get`

```tsx
const readOnlyAtom = atom(get => get(priceAtom) * 2)
```

- 쓰기 전용 아톰(Write-only): 상태를 변경하기 위한 아톰
  - 첫 번째 인자: 초기 상태값 설정
  - 두 번째 인자: 상태 수정 update 로직
  - 파생 아톰의 변경이 부모 아톰의 변경이 서로 영향을 주고받음

```tsx
const writeOnlyAtom = atom(
  null, // 초기 상태값. 보통 null로 지정 (컨벤션)
  (get, set, update) => {
    // 두 가지 방식의 상태 수정 로직
    // get은 다른 아톰의 값을 읽을 수 있고, 여기에서의 get은 값을 추적하지 않음
    // set을 사용해 상태를 수정하는데, 여기서 priceAtom의 값을 변경
    set(priceAtom, get(priceAtom) - update.discount) // (1) priceAtom에서 현재 값을 읽고 discount를 차감하여 업데이트
    set(priceAtom, price => price - update.discount) // (2) 함수형 업데이트 (현재 값을 인자로 받아 변경)
  },
)
```

- 읽기-쓰기 아톰(Read-Write atom): 읽기와 쓰기 모두 가능한 아톰

```tsx
const readWriteAtom = atom(
  get => get(priceAtom) * 2, // read
  (get, set, newPrice) => {
    set(priceAtom, newPrice / 2)
  }, // write
)
```

> 이때, read 함수의 `get`과 쓰기 전용 아톰에서의 `get`의 역할이 다르다.
>
> - `read` 함수의 `get`: 아톰 값을 읽고, 그 값을 추적하여 리렌더링
> - `write` 함수의 `get`: 아톰 값을 읽지만, 값을 추적하지 않음
> - `write` 함수의 `set`: 변경하고자 하는 아톰을 업데이트

## Jotai가 아톰 값을 저장하는 방식 이해하기

- Jotai에서는 아톰 구성이 직접 해당 값을 가지지 않음
- 아톰 값을 저장하는 `store`가 따로 있음
  - `store`: 키가 아톰 구성 객체, 값이 아톰 값인 `WeakMap` 객체
- `useAtom`을 사용하면 기본적으로 모듈 수준에서 정의된 기본 `store` 사용 → 컴포넌트에서 `store`를 생성하려면?

#### Jotai의 `Provider` 컴포넌트로 컴포넌트 레벨에서 `store` 생성하기

```tsx
import { atom, useAtom, Provider } from 'jotai'

const countAtom = atom(0)

const Counter = ({ countAtom }) => {
  const [count, setCount] = useAtom(countAtom)
  const inc = () => setCount(c => c + 1)
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  )
}

const App = () => (
  <>
    <Provider>
      <h1>First Provider</h1>
      <div>
        <Counter />
      </div>
      <div>
        <Counter />
      </div>
    </Provider>
    <Provider>
      <h1>Second Provider</h1>
      <div>
        <Counter />
      </div>
      <div>
        <Counter />
      </div>
    </Provider>
  </>
)
```

- `Provider`에 `value` prop를 따로 제공하지 않아도 자동으로 아는 이유?
  - `Provider` 내부적으로 `StoreContext.Provider`를 만들어 사용하고 여기서 `value` prop에 `store || storeRef.current`로 넘겨주고 있음
  - Jotai의 아톰은 실제 값을 가지고 있지 않음 → 단지 상태의 “설명” 역할만
  - 아톰 값을 저장하고 있는 `store` 객체가 따로 있음 → 키가 아톰 구성 객체, 값이 아톰 값인 `WeakMap`
  - `store`에서 각각의 아톰 상태를 독립적으로 저장하고 실제 상태와 매핑

```tsx
// https://github.com/pmndrs/jotai/blob/main/src/react/Provider.ts
export const Provider = ({
  children,
  store,
}: {
  children?: ReactNode
  store?: Store
}): ReactElement<{ value: Store | undefined }, FunctionComponent<{ value: Store | undefined }>> => {
  const storeRef = useRef<Store>(undefined)
  if (!store && !storeRef.current) {
    storeRef.current = createStore()
  }
  return createElement(
    StoreContext.Provider,
    {
      value: store || storeRef.current,
    },
    children,
  )
}
```

## Atoms-in-Atom

#### Jotai에서 배열을 아톰으로 다룰 때 고민해볼 사항

- 단일 요소를 변경하기 위해 todos 배열 전체를 갱신해야 함
- 요소의 id 값 → 주로 map에서 key로 사용되는데, id를 사용하지 않는 것이 좋다

#### Jotai 멘탈 모델에 적합한 Atoms-in-Atom

- `atoms-in-atom`에서는 개별 `todo`가 독립적으로 관리 → 배열 전체를 복사할 필요 X
- id 자체를 유지할 필요가 없는 `atoms-in-atom` 구조
  - 리액트에는 배열 요소를 렌더링할 때 `key`로 사용하는 값이 변경되면, 요소를 재사용하지 않고 새로 렌더링
  - id를 key로 사용하는 경우 배열의 요소가 자주 변경되거나 재배치될 때, 불필요한 DOM 업데이트가 발생할 가능성 ([key를 이용해 폼을 초기화하기](https://ko.react.dev/learn/preserving-and-resetting-state#resetting-a-form-with-a-key))
  - `atoms-in-atom`에서는 Atom 객체 자체를 사용 가능 → key 안정성 보장

```tsx
const countsAtom = atom([atom(1), atom(2), atom(3)])

const Counter = ({ countAtom }) => {
  const [count, setCount] = useAtom(countAtom)
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </div>
  )
}

const Parent = () => {
  const [counts, setCounts] = useAtom(countsAtom)
  const addNewCount = () => {
    const newAtom = atom(0)
    setCounts(prev => [...prev, newAtom])
  }
  return (
    <div>
      {counts.map(countAtom => (
        <Counter countAtom={countAtom} key={countAtom} />
      ))}
      <button onClick={addNewCount}>Add</button>
    </div>
  )
}
```

- 유니크한 key라면 `key={countAtom}`으로 쓸 수도 있을텐데 `key={`${countAtom}`}`으로 쓰는 이유?
  - 타입스크립트에서는 key가 문자열 타입이어야만 하기 때문에 문자열로 변경해서 작성

#### Atoms-in-Atom 패턴을 사용했을 때의 차이점

- 배열 아톰은 아톰이 요소인 배열을 보관하는 데 사용
- 배열에 새로운 요소를 추가하려면 새로운 아톰을 생성해서 추가해야 함
- 아톰 구성은 문자열로 평가할 수 있으며 UID를 반환
- 요소를 렌더링하는 컴포넌트는 각 컴포넌트에서 아톰 요소 사용 → 독립적인 요소로 관리, 리렌더링 최적화
