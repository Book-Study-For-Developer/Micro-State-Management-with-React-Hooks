# 02. 지역 상태와 전역 상태 관리하기

depth가 깊지 않다면 상위에서 상태를 정의하고 쓰는게 바람직하다

하지만 depth가 깊어지면 관리가 힘들다

전역상태를 사용한다면 범위를 어느정도 하고 어떤 상태를 공유하는지 설계하는 것이 중요

싱글톤 패턴을 쓰기보다는 컨테이너를 하나 만들어서 쓰는 것이 모듈화된 방식이다.

전역 상태를 쓸 때는 관리를 잘해야 한다. 어디서든 접근 가능하기에

멱등성 -> API 통신의 개념에서만 생각했고 단순하게 순간적으로 몰려있는 요청에 대해 중복처리를 방지하고 하나의 결과값을 뱉는? 그런 모호하게 알고 있었던 개념을 이번에 좀 더 자세히 알 수 있었다.

멱등성(Idempotent)이란 키워드는 토스의 아래 포스트를 통해 처음 접했다
[멱등성이 뭔가요](https://docs.tosspayments.com/blog/what-is-idempotency)

HTTP 메서드에 따른 멱등성

| Method  | Safe | Idempotent | Reference                                                             |
| ------- | ---- | ---------- | --------------------------------------------------------------------- |
| CONNECT | no   | no         | [Section 4.3.6](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.6) |
| DELETE  | no   | yes        | [Section 4.3.5](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.5) |
| GET     | yes  | yes        | [Section 4.3.1](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.1) |
| HEAD    | yes  | yes        | [Section 4.3.2](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.2) |
| OPTIONS | yes  | yes        | [Section 4.3.7](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.7) |
| POST    | no   | no         | [Section 4.3.3](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.3) |
| PUT     | no   | yes        | [Section 4.3.4](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.4) |
| TRACE   | yes  | yes        | [Section 4.3.8](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.8) |

멱등성 -> 리소스를 변경 해도 같은 리소스로 업데이트 됨
안전성 -> 리소스를 변경하지 않음

안전성이 있는 메소드는 멱등하다
멱등성이 있는 메소드는 안전하지 않을 수 잇다.

---

## 지역 상태를 효과적으로 사용하는 방법

두 가지 방법이 있다.

1. lifting state up
2. lifting content up

### lifting state up - 상태 끌어올리기

- 두 컴포넌트에서 사용하는 동일한 역할의 상태를 하나로 공유하기 위해 이들을 감싸는 부모 컴포넌트를 만들고 자식 컴포넌트에게 상태를 props로 내려준다.
- 전형적으로 보이는 패턴이다.
- 이는 자식 컴포넌트들이 동일하게 리렌더링이 발생하기 때문에 의도된 형태가 아니라면 성능의 저하를 유발한다.

```tsx
const ParentComponent = () => {
  const [value, setValue] = useState('');
  const handleChange = (newValue: string) => {
    setValue(newValue);
  };

  return (
    <>
      <ChildComponent1 value={value} onChange={handleChange} />
      <ChildComponent2 value={value} onChange={handleChange} />
    </>
  );
};

type ChildComponentProps = {
  value: string;
  onChange: (newValue: string) => void;
};

const ChildComponent1 = ({ value, onChange }: ChildComponentProps) => {
  return (
    <input
      type="text"
      value={value}
      onChange={(e) => onChange(e.target.value)}
    />
  );
};

const ChildComponent2 = ({ value, onChange }: ChildComponentProps) => {
  return (
    <input
      type="text"
      value={value}
      onChange={(e) => onChange(e.target.value)}
    />
  );
};
```

### lifting content up - 내용 끌어올리기

- 컴포넌트가 다양하고 복잡하게 쪼개져 있다면 불필요한 렌더링이 발생할 수 있다.
- A-B-C 순으로 depth가 발생하는 구조일 떄 A에서 정의한 상태가 B에서 써야한다면 C의 경우 상태가 변화가 없음에도 부모가 리렌더링됨으로써 같이 리렌더링 된다.
- 이를 방지하기 위한 방법이다.
- 상태를 정의한 부모 컴포넌트를 감싸는 컴포넌트를 하나 더 만들고 해당 컴포넌트에서 부모 컴포넌트로 불필요한 리렌더링이 발생했던 컴포넌트를 props로 넘겨준다.
- props로 넘겨진 컴포넌트는 상태가 정의된 컴포넌트보다 상위에서 오기 때문에 리렌더링의 대상이 되지 않는다.

```tsx
const Independent = () => {
  return <p>Don't touch me</p>;
};

const SuperComponent = () => {
  {/* ===== ADDED ===== */}
  return <ParentComponent child={<Independent />} />;
};

type ParentComponentProps = {
  child: React.ReactNode;
};

const ParentComponent = ({ child }: ParentComponentProps) => {
// ...... 생략
  return (
    <>
      <ChildComponent1 value={value} onChange={handleChange} />
      {/* ===== ADDED ===== */}
      <ChildComponent2 value={value} onChange={handleChange} child={child} />
    </>
  );
};

type ChildComponentProps = {
  value: string;
  onChange: (newValue: string) => void;
};

const ChildComponent1 = ({ value, onChange }: ChildComponentProps) => {
  // ....... 생략
};

const ChildComponent2 = ({
  value,
  onChange,
  child, {/* ===== ADDED ===== */}
}: ChildComponentProps & { child: React.ReactNode }) => {
  return (
    <>
      {child} {/* ===== ADDED ===== */}
     // ...... 생략
    </>
  );
};
```

- children props가 우리에게는 익숙한 형태일 텐데 lifting content up을 변형한 형태이다.

```tsx
const SuperComponent = () => {
  return (
    <ParentComponent>
      <Independent /> {/* wrapper 형태로 변경 */}
    </ParentComponent>
  );
};

type ParentComponentProps = {
  children: React.ReactNode; children으로 변경
};

const ParentComponent = ({ children }: ParentComponentProps) => {
  // ...... 생략
  return (
    <>
      <ChildComponent2 value={value} onChange={handleChange}>
        {children} {/* ChildComponent2로 자식을 넘긴다. */}
      </ChildComponent2>
    </>
  );
};

const ChildComponent2 = ({
  value,
  onChange,
  children, // children로 변경
}: ChildComponentProps & { children: React.ReactNode }) => {
  return (
    <>
      {children} {/* children 렌더링 */}
      // ...... 생략
    </>
  );
};
```

## 전역 상태 사용하기

- 지역 상태가 아닌 것을 전역 상태라고 한다.
  - 하나의 컴포넌트에서 쓰고 있고 캡슐화된 형태 -> 지역 상태
  - 여러 컴포넌트에서 쓰고 있는 값 -> 전역 상태
- 전역 상태를 사용해야 할 떄는 ?
  - props로 전달하는 것이 적절치 않을 때
  - 리액트 외부에 상태가 있을 때
- 특정 스코프에서 상태를 공유 -> 전역으로 볼것인가? 지역으로 볼것인가?
