## 02. 지역 상태와 전역 상태 사용하기

> 순수 함수는 오직 인수에만 의존하며 동일한 인수를 받은 경우 동일한 값을 반환한다. 상태는 인수 외부의 값을 말하며 상태에 의존하는 함수는 순수하지 않게 된다.

```js
const addOne = (n) => n + 1;
```
addOne 함수의 경우에는 순수함수이다.


```js
let base = 1;

const addBase = (n) => n + base;
```

base의 변경되지 않는다면 순수함수지만 외부변수에 의존하기 때문에 순수함수가 아닐 수 있다.


```js
const createContainer = () => {
  let base = 1;
  const addBase = (n) => n + base;
  const changeBase = (b) => { base = b; };
  return { addBase, changeBase };
};

const { addBase, changeBase } = createContainer();
```

모듈화된 패턴을 활용해서 재사용성을 늘리고 컨테이너들을 격리시켰다.

### 지역 상태의 한계

> 함수 컴포넌트 외부에서 상태를 변경해야한다면 저녁 상태가 필요하다. 전역 상태는 컴포넌트 외부에서 리액트 컴포넌트의 동작을 제어할 때 유용하게 사용할 수 있지만 컴포넌트 동작을 예측하기 어렵다는 장단점이 있다.

### 지역 상태를 효과적으로 사용하는 방법

- 상태 끌어올리기 ( Lifting State Up )
- 내용 끌어올리기 ( Lifting Content Up )

#### 상태 끌어올리기

```jsx
const Component1 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>
        Increment Count
      </button>
    </div>
  );
};

const Component2 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>
        Increment Count
      </button>
    </div>
  );
};

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

부모 컴포넌트로 지역 상태인 count useState 값을 올려 자식에게 전달한다. **단, 상태를 상위로 전달할 경우 Parent는 모든 자식 컴포넌트를 포함해 하위 트리 전체를 리렌더링 한다.**


#### 내용 끌어올리기

```js
const Component1 = ({ count, setCount, additionalInfo }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>
        Increment Count
      </button>
      {additionalInfo}
    </div>
  );
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

const GrandParent = () => {
  return <Parent additionalInfo={<AdditionalInfo />} />;
};
```

이 패턴은 사실 처음 보는 패턴인데 자식 컴포넌트에서 직접 선언하면 state가 변경됨에 따라 불필요한 렌더링이 발생할 수 있기 때문에 상위 컴포넌트에서 주입시켜주는 방식으로 불필요한 렌더링을 제거해주는 패턴인 것 같다. 

>❓ 익숙하지 않아서 인지 해당 패턴은 오히려 컴포넌트의 흐름을 더 복잡하게 만드는 것 같다.

위와 같은 방식보다 아래와 같은 **children을 이용한 합성 컴포넌트**를 만드는것이 좋아보인다.

```js
const AdditionalInfo = () => {
  return <p>Some information</p>
};

const Component1 = ({ count, setCount, children }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>
        Increment Count
      </button>
      {children}
    </div>
  );
};

const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return (
    <>
      <Component1 count={count} setCount={setCount}>
        {children}
      </Component1>
      <Component2 count={count} setCount={setCount} />
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

### 언제 전역 상태를 사용할까?

- prop을 전달하는것이 적절하지 않을떄 
- 이미 리액트 외부에 상태가 있을때 

#### prop을 전달하는것이 적절하지 않을 때

```js
const Component1 = ({ count, setCount }) => {
  return (
    <div>
      {count}
      <button onClick={() => setCount((c) => c + 1)}>
        Increment Count
      </button>
    </div>
  );
};

const Parent = ({ count, setCount }) => {
  return (
    <>
      <Component1 count={count} setCount={setCount} />
    </>
  );
};

const GrandParent = ({ count, setCount }) => {
  return (
    <>
      <Parent count={count} setCount={setCount} />
    </>
  );
};

const Root = () => {
  const [count, setCount] = useState(0);
  return (
    <>
      <GrandParent count={count} setCount={setCount} />
    </>
  );
};

```

위와 같은 경우는 상태가 변경되면 하위 모든 트리가 변경되고 불필요하게 중간 컴포넌트에 prop을 전달해야한다. **전역 상태를 사용하면 사용하는 컴포넌트에 직접 주입할 수 있다.** 

#### 이미 리액트 외부에 상태가 있을 때

서버로부터 저장되어 있는 값을 전달받아서 여러 컴포넌트에서 사용할때를 말하는 것 같다. ( ex. 사용자 인증 정보 ) 

```js
const globalState = {
  authInfo: { name: 'React' },
};

const Component1 = () => {
  // useGlobalState is a pseudo hook
  const { authInfo } = useGlobalState();
  return (
    <div>
      {authInfo.name}
    </div>
  );
};
```



