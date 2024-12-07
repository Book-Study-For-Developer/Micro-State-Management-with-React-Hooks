### 멱등성이란?

책(p31)에서 멱등성을 changeBase를 통해 base를 변경하지 않는 이상 addBase를 호출하면 동일한 결과를 얻을 수 있는 특성이라고 소개한다.

멱등(Idempotent)하다는 것은 첫 번째 수행을 한 뒤 여러 차례 적용해도 결과를 변경시키지 않는 작업을 뜩한다.

HTTP 메서드에서의 멱등성

| 메서드 | 멱등성 |
| ------ | ------ |
| GET    | O      |
| PUT    | O      |
| POST   | X      |
| PATCH  | X      |
| DELETE | O      |

지역 상태와 멱등의 예시

```tsx
let base = 1;

const addBase = (n: number) => n + base;

// useState를 사용한 예시
const AddBase = ({ number }: { number: number }) => {
  const [base, changeBase] = useState(1);
  return <div>{number + base}</div>;
};
```

## 지역상태를 효과적으로 사용하는 방법

여기서 말하는 ‘효과적’은 여러 컴포넌트에서 지역상태를 사용하지만 같은 상태를 바라봐야 하는 상황을 말하고 있다. 그렇게 사용하고 싶을 때 쓰는 방법이 2가지가 있다.

1. 상태 끌어올리기(`lifting state up`)
2. 내용 끌어올리기(`lifting content up)`

### 상태 끌어올리기

컴포넌트 1, 2에서 하나의 상태를 바라보기 위해서는 그것을 감싼 부모 컴포넌트를 생성해서 사용하면 된다.

```tsx
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

여기서 조심해야 할 점은 두 개의 컴포넌트 모두 리렌더링이 일어난 다는 점이다.

### 내용 끌어올리기

상태 끌어올리기를 했지만 전혀 관계가 없는 컴포넌트가 리렌더링하는 문제가 발생할 수 있다.

아래의 코드에서는 `AdditionalInfo` 가 해당되는 컴포넌트로 볼 수 있다.`count`가 바뀌면 불필요하게 리렌더링이 일어날 수 있는 것이다.

그렇기에 children props를 사용한다면 그러한 문제를 해결할 수 있다.

```tsx
const AdditionalInfo = () => {
  return <p>Some information</p>;
};

const Parent = ({ additionalInfo }) => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1
        count={count}
        setCount={setCount}
        additionalInfo={additionalInfo}
      />
      <Component2 count={count} setCount={setCount} />
    </>
  );
};

// props로 넘겨주는 방식
const GrandParent = () => {
  return <Parent additionalInfo={<AdditionalInfo />} />;
};

// 또는 children으로 넘겨주는 방식
const GrandParent = () => {
  return (
    <Parent>
      <AdditionalInfo />
    </Parent>
  );
};
```

## 전역 상태에 대해서

전역상태가 만족하는 **두 가지 특성**이 있다.

- 싱글턴: 특정 컨텍스트에서 상태가 하나의 값을 가지고 있다는 것을 의미
  - Next + tanstack-query를 쓴다면 서버와 브라우저에서 서로 다른 컨텍스트로 관리해야 한다.(https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr#server-components--nextjs-app-router)
- 공유 상태: 상태 값이 다른 컴포넌트 간에 공유된다.

### 언제 전역 상태를 사용하는 것이 좋을까?

- props drilling이 심할 때 ⇒ 누구나 흔히 아는 이유다.
- 이미 리액트 외부에 상태가 있을 때 ⇒ 외부에 리액트와 라이프사이클과 상관없는 상태를 의미한다.
  ```tsx
  // 외부에 존재하는 상태 값
  const globalState = {
    authInfo: { name: "React" },
  };

  const Component1 = () => {
    // useGlobalState is a pseudo hook
    const { authInfo } = useGlobalState();

    return <div>{authInfo.name}</div>;
  };
  ```
