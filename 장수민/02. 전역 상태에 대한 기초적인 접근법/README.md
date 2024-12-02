## 2. 전역상태에 대한 기초적인 접근법

2부에서 주요하게 다룰 내용은 다음과 같다.

- 지역 상태와 전역 상태 사용하기
- 리액트 컨텍스트를 이용한 컴포넌트 상태 공유
- 구독을 이용한 모듈 상태 공유
- 리액트 컨텍스트와 구독을 이용한 컴포넌트 상태 공유

### 지역 상태와 전역 상태 사용하기

리액트 컴포넌트는 트리 구조를 구성하는데, 📌 특정 상황에서 트리 내 서로 멀리 떨어져 있는 둘 이상의 컴포넌트에 공통적인 상태가 필요한 경우가 있다.

이러한 경우 전역 상태가 필요하고, 개념적으로는 특정 컴포넌트에 속해 있지 않으므로 전역 상태를 저장하는 **위치**를 고려해야 한다.

### 언제 지역 상태를 사용할까?

선언된 상태가 컴포넌트 내에서만 사용된다면 다른 컴포넌트에 영향을 미치지 않게 되고 이를 책에서 '억제됨(contained)'이라고 표현한다.

> [!TIP]
> 자바스크립트 함수는 순수/비순수 함수로 나눌 수 있다. 순수 함수는 오직 인수에만 의존하면서 동일한 인수를 받은 경우 동일한 값을 반환한다. useEffect를 사용한 컴포넌트인 경우 비순수 함수에 해당한다.

```javascript
let base = 1;

const addBase = (n) => n + base;
```

외부에서 함수 작동 방식을 변경할 수 있다는 점을 강점으로 볼 수 있지만, `addBase` 함수가 외부 변수에 의존한다는 사실을 모르고 다른 곳에 사용될 수 있다는 점은 단점이다.

```tsx
const createContainer = () => {
  let base = 1;
  const addBase = (n) => n + base;
  const changeBase = (b) => {
    base = b;
  };
  return { addBase, changeBase };
};
```

이제는 base(window.base)가 싱글톤이 아니기 때문에 코드의 재사용성을 높일 수 있고, createContainer 함수를 사용해서 독립된 컨테이너를 생성하면 다른 컨테이너에 영향을 주지 않고 사용할 수 있다.

> [!NOTE]
> 왜 싱글톤을 사용하면 재사용성이 떨어진다고 얘기할까? -> 많은 개발자가 전역 변수가 예기치 않은 시점에 변경될 우려를 하고 있기 때문에 싱글톤인 전역 변수를 선언하였어도, 결국 잘 사용하지 않는다는 것을 의미하는 걸까...❓

### 지역 상태를 효과적으로 사용하는 방법

**상태 끌어올리기(Lifting State Up)**

```tsx
const Component1 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>Increment Count</button>
    </div>
  );
};

// Component2는 Compoenent1와 구현 동일
// const Component2 = ...

const Parent = () => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1 count={count} setCount={setCount} />
      <Component2 count={count} setCount={setCount} />
    </>
  );
};
```

count 상태는 Parent에서 단 한 번만 정의되고 Component1, Component2에서 공유된다. count는 여전히 컴포넌트 내에 지역 상태로 존재한다.

> [!IMPORTANT]
> 지역 상태를 사용하는 대부분의 상황에서 작동하지만 성능 문제가 생길 수 있다. 상태를 상위 컴포넌트로 전달할 경우 Parent는 모든 자식 컴포넌트를 포함해 하위 트리 전체를 리렌더링할 것이다.

**내용 끌어올리기(Lifting Content Up)**

```tsx
const AdditionalInfo = () => {
  return <p>Some information</p>;
};
```

위 컴포넌트가 Component1의 하위에 들어있을 때, count가 변경되면 Parent가 리렌더링된 후에 Component1, Component2, AdditionalInfo도 모두 리렌더링된다.

그런데 AdditionalInfo는 count에 의존하지 않기 때문에 리렌더링을 막을 필요가 있다. 이러한 경우에 GrandParent 요소를 만들고 AdditionalInfo 요소를 상위 컴포넌트로 끌어올릴 수 있다.

### 전역 상태 사용하기

상태가 하나의 컴포넌트에만 속하지 않고 여러 컴포넌트에서 사용할 수 있다면 전역 상태라고 한다. 모든 컴포넌트가 의존하는 애플리케이션 차원의 지역 상태가 있을 수 있는데, 이 경우는 전역 상태라고 볼 수 있다.

> [!NOTE]
> 전역 상태를 싱글톤으로 둘 수도, 메모리상에 여러 값을 둘 수도 있다. ❓ 이 중 어떤 방법이 더 나은지에 대해 고민해보는 게 유의미한지는 잘 모르겠다...

```tsx
// 싱글톤이 아닌 전역 상태
const createContainer = () => {
  let base = 1;
  const addBase = (n) => n + base;
  const changeBase = (b) => {
    base = b;
  };
  return { addBase, changeBase };
};

const container1 = createContainer();
const container2 = createContainer();

container1.changeBase(10);
console.log(container1.addBase(2)); // 12
console.log(container2.addBase(2)); // 3
```

사용을 고려해볼 만한 시점은 다음과 같다.

1. prop을 전달하는 것이 적절하지 않을 때(혹은, props drilling이 심하게 일어날 때라고 말할 수 있겠다.)
2. 이미 리액트 외부에 상태가 있을 때
