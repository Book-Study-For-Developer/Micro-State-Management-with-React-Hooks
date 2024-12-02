# 2. 지역상태와 전역 상태 사용하기

## 언제 지역 상태를 사용할까.

- **순수 함수** : 인수에만 의존하며 동일한 인수를 받은 경우 동일한 값을 반환한다.
- **비순수 함수** : 리액트의 경우 상태를 활용하면 순수하지 않게 된다.
  → 하지만 상태가 컴포넌트 내에만 있다면 다른 컴포넌트에 영향을 미치지 않는다. 이 상태를 억제됨(contained)라고 표현한다.

### 순수 vs 멱등성

제가 알고 있는 순수함수의 ‘순수’와 멱등성의 의미가 동일한 입력에 대해 동일한 결과를 반환한다는 의미로 인지하고 있어서 실제로 같은 의미로서 사용되는지 궁금해서 알아보았습니다.

1. **순수함수**

   - Input에 대한 Output이 항상 동일한 함수
   - 외부 상태를 변경하지 않고, 의존하지 않음
   - 부수효과 (side effect)가 없음

   ex)

   ```tsx
   // 순수 함수
   function add(a, b) {
     return a + b; // 입력 a, b에만 의존, 외부에 영향을 주지 않음
   }

   console.log(add(2, 3)); // 항상 5를 반환
   console.log(add(2, 3)); // 항상 5를 반환 (동일한 입력에 동일한 결과)
   ```

2. **멱등성**

   - 같은 연산을 여러 번 해도 결과가 변하지 않는 특성
     -> 멱등성은 반드시 **같은 결과를 반환하지 않아도** 됨. **결과가 변하면 안됨.**
   - 함수가 연산을 여러 번 호출해도 외부 상태에 변화가 없음
   - 주로 외부 상태를 다루는 연산에서 사용됨 ( ex: HTTP 메서드, 데이터베이스 업데이트 )

   ex)

   ```jsx
   // 같은 입력이 들어오면 같은 외부 상태가 유지됨 (상태 변화를 여러 번 해도 동일함)
   function setBackgroundColor(color) {
     document.body.style.backgroundColor = color;
   }

   setBackgroundColor("red"); // 배경색을 빨간색으로 변경
   setBackgroundColor("red"); // 배경색은 여전히 빨간색 (결과는 동일)
   ```

   - **HTTP 요청의 멱등성 예시:**
     - **GET**: 여러 번 호출해도 동일한 데이터를 반환 → 멱등적.
     - **PUT**: 동일한 리소스를 업데이트하면 결과가 항상 동일함 → 멱등적.
     - **POST**: 동일한 요청을 여러 번 보내면 새로운 리소스가 계속 생성됨 → 멱등적이지 않음.

매 시도에 대해서 같은 결과가 나온다는 점에서 ‘순수’와 ‘멱등성’은 비슷한 점이 있었습니다. 하지만 ‘순수’와는 다르게 ‘멱등성’은 외부 상태를 변경이 가능하다는 점에서 차이가 있었습니다. 의미상 순수가 멱등성보다 좁은 의미로 활용되는 것 같습니다.

<br />

## 지역 상태를 효과적으로 사용하는 방법

- 상태 끌어올리기(lifting state up) 패턴  
  → 공통으로 사용하는 상태를 부모 컴포넌트에서 정의하고, 자식 컴포넌트로 공유하는 방식

  ```jsx
  const Comp1 = ({ count, setCount }) => {
    return <div> ... </div>;
  };

  const Comp2 = ({ count, setCount }) => {
    return <div> ... </div>;
  };

  const Parent = ({}) => {
    const [count, setCount] = useState(0);
    return (
      <>
        <Comp1 count={count} setCount={setCount} />
        <Comp2 count={count} setCount={setCount} />
      </>
    );
  };
  ```

  → Parent가 리랜더링되면 하위 트리 전체를 리랜더링한다.

<br />

- 내용 끌어올리기(lifting content up) 패턴  
  → 불필요한 리랜더링 방지하기 위해 JSX 요소를 상위 컴포넌트로 끌어올리는 방식

  ```jsx
  const AdditionalInfo = () => {
    return <p> Some Info... </p>;
  };

  const Comp1 = ({ count, setCount, children }) => {
    return (
      <div>
        ...
        {children}
        ...
      </div>
    );
  };

  const Parent = ({ children }) => {
    const [count, setCount] = useState(0);

    return (
      <>
        <Comp1 count={count} setCount={setCount}>
          {children}
        </Comp1>
      </>
    );
  };

  const GrandParent = () => {
    return (
      <Parent>
        <AdditionalInfo />
      </Parent>
    );
  };
  ```

  → `AdditionalInfo` 컴포넌트는 count 상태나 props을 가지지 않기 때문에 Virtual DOM에서 정적 요소라고 판단하고 재사용한다.

<br />

## 전역상태 사용하기

**지역 상태** - 개념적으로 하나의 컴포넌트에 속하고 컴포넌트에 의해 캡술화된 상태, 특정 컨텍스트에서 상태가 하나의 값을 가지고 있다는 것.  
**전역 상태** - 컴포넌트에만 속하지 않고 여러 컴포넌트에서 사용할 수 있는 상태.

→ 지역 상태와 전역 상태는 어떤 위치에서 보느냐에 따라 구분이 모호할 수 있다.  
→ 개념적으로 속한 곳이 어디인지 생각해보면 어떤 상태인지 알 수 있다.  
→ 전역 상태에 대해서는 두가지 측면을 고려한다.

- **싱글턴** → 컨텍스트에서 상태가 하나의 값을 가지고 있는 것
- v공유 상태\*\* → 상태 값이 다른 컴포넌트 간에 공유된다는 것을 의미. 하지만 JS 메모리 상 단일 값일 필요는 없다. 싱글턴이 아닌 전역 상태에서는 여러 값을 가질 수 있음.

### 전역 상태를 사용해할 때

- props로 전달하는 것이 적절하지 않을 때  
  → props를 전달하는 과정이 번거롭거나, 상태 변경 시, props를 UI로직에서 사용하지 않는 중간 컴포넌트들도 모두 리랜더링 되면서 성능상 불이익이 있을 수 있음.
- 이미 상태가 외부에 정의되어 있을 떄 ( ex. 사용자 인증 정보 )
