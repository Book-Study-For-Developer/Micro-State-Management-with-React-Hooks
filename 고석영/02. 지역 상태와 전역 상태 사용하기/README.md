## 언제 지역 상태를 사용할까?

### 순수 함수

- 오직 인수에만 의존하며 같은 인수에 대해 항상 같은 값을 반환하는 함수

```tsx
const addOne = n => n + 1
```

> 리액트 컴포넌트도 마찬가지로 자바스크립트 함수이기 때문에 순수할 수 있지만 컴포넌트 내에서 상태를 사용할 경우 순수하지 않게 된다. 그러나 상태가 컴포넌트 내에서만 사용된다면, 다른 컴포넌트에 영향을 미치지 않는다. → 억제됨

### 컨테이너 객체

- 싱글턴을 사용하는 대신 컨테이너 객체를 만드는 것이 더 모듈화된 접근 방식 → 격리된 컨테이너
- 컨테이너가 격리되었다? → 재사용은 더 쉬워지고, 각 컨테이너를 다른 컨테이너에 영향을 주지 않고 사용할 수 있음!

```tsx
const createContainer = () => {
  let base = 1
  const addBase = n => n + base
  const changeBase = b => {
    base = b
  }
  return { addBase, changeBase }
}

const { addBase, changeBase } = createContainer()
```

> 컨테이너의 `addBase`가 수학적으로 순수한 함수는 아니지만, `changeBase`를 통해 `base`를 변경하지 않는 이상 `addBase`를 호출하면 항상 동일한 결과를 얻을 수 있다. 이같은 특성을 **멱등성**이라고 부르기도 한다.

> [!NOTE]
>
> #### 멱등성
>
> - 첫 번째 수행을 한 뒤 여러 차례 적용해도 결과를 변경시키지 않는 작업 또는 기능의 속성
> - `fn(fn(x)) ≡ f(x)` 인 경우 멱등 법칙을 만족 → 예) 멱등성을 가진 절대값 > 함수 `abs(abs(x)) ≡ abs(x)`
> - 보통 HTTP 메서드에서 많이 사용되는 개념(HTTP 공부하면서 처음 알게 되었음..)

### 리액트 컴포넌트

- 리액트의 컴포넌트: `props`를 인수로 받아 JSX 요소 반환 → 기본적으로 순수 함수

```tsx
const AddOne = ({ number }) => {
  return <div>{number + 1}</div>
}
```

### 지역 상태와 `useState`

```tsx
const AddBase = ({ number }) => {
  const [base, changeBase] = useState(1)

  return <div>{number + base}</div>
}
```

- 인수에 포함되지 않은 `base`에 의존하게 되면서 엄밀히 말해 순수하지 않게 됨

> `useState`는 변경되지 않는 한 `base`를 반환하므로 `AddBase`는 멱등성을 지닌다?

- 초기값인 `1`로 `base`가 설정되고, 이후 `base` 값을 변경하지 않으면 그 값은 계속 유지
- `AddBase` 컴포넌트가 다시 렌더링될 때마다, `number + base`가 계산되는데 `base`가 변경되지 않는 한 같은 값을 계속 반환

> 컴포넌트는 억제돼 있고 컴포넌트 외부의 그 어떤 것에도 영향을 미치지 않기 때문에 지역성을 보장한다.

- `useState`가 포함된 `AddBase`는 `changeBase`를 함수 선언 범위 내에서만 사용할 수 있음 → 컴포넌트는 억제됨
- 즉, 함수 밖에서 `base` 변경 불가능 → 지역성을 보장

## 지역 상태를 효과적으로 사용하는 방법

### 상태 끌어올리기(Lifting State Up)

- 두 컴포넌트가 하나의 상태로부터 영향을 받을 때, 공통의 상위 컴포넌트 트리에서 상태를 정의하는 방법

```tsx
const Parent = () => {
  const [count, setCount] = useState(0)

  return (
    <>
      <Component1 count={count} setCount={setCount} />
      <Component2 count={count} setCount={setCount} />
    </>
  )
}
```

### 내용 끌어올리기(Lifting Content Up)

- 불필요한 렌더링을 방지하기 위해 JSX 요소를 상위 컴포넌트로 끌어올리는 방법
- `children` prop → 부모 컴포넌트가 변경되더라도 `children` prop으로 전달된 자식 컴포넌트는 리렌더링하지 않음

```tsx
const AdditionalInfo = () => {
  return <p>Some information</p>
}

const Component1 = ({ count, setCount, children }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount(c => c + 1)}>Increment Count</button>
      {children}
    </div>
  )
}

const Parent = ({ children }) => {
  const [count, setCount] = useState(0)
  return (
    <>
      <Component1 count={count} setCount={setCount}>
        {children}
      </Component1>
      <Component2 count={count} setCount={setCount} />
    </>
  )
}

const GrandParent = () => {
  return (
    <Parent>
      <AdditionalInfo />
    </Parent>
  )
}
```

이 코드를 바벨로 돌려보면,

```tsx
const GrandParent = () => {
  return /*#__PURE__*/ _jsx(Parent, {
    children: /*#__PURE__*/ _jsx(AdditionalInfo, {}),
  })
}
```

- AdditionalInfo가 Parent가 아닌 GrandParent에서 생성되어 전달
- 즉, AdditionalInfo가 Parent에 의존적이지 않음 → 부모인 Parent가 변경되더라도 리렌더링이 일어나지 않는 이유

## 전역 상태 사용하기

#### 전역 상태

- 상태가 하나의 컴포넌트에만 속하지 않고 여러 컴포넌트에서 사용할 수 있다면 전역 상태
- 전역 상태의 두 가지 측면
  - 싱글턴이며, 특정 컨텍스트에서 상태가 하나의 값을 가지고 있다는 것을 의미
  - 공유 상태이며, 상태 값이 다른 컴포넌트 간에 공유된다는 것을 의미
- 싱글턴이 아닌 전역 상태 → 격리된 각각의 컨테이너

```tsx
const createContainer = () => {
  let base = 1
  const addBase = n => n + base
  const changeBase = b => {
    base = b
  }
  return { addBase, changeBase }
}

const container1 = createContainer()
const container2 = createContainer()

container1.changeBase(10)

console.log(container1.addBase(2)) // 12
console.log(container2.addBase(2)) // 3
```

### 전역 상태를 사용하는 경우

#### prop을 전달하는 것이 적절하지 않을 때

- 여러 단계로 구성된 컴포넌트를 통해 props를 전달 → props drilling
  - 좋지 못한 개발자 경험
  - 상태 변경 시 중간 컴포넌트들이 리렌더링 되며, 성능 문제 발생

#### 이미 리액트 외부에 상태가 있을 때

- 인증 정보 등 리액트 외부에서 가져오는 전역 상태 → 이때, 리액트로 연결해주는 작업 필요
