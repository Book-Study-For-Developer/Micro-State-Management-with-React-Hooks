## 시나리오 2: Jotai

### Jotai 알아보기

```jsx
const Counter1 = () => {
  const [count , setCount] = useState(0);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}

const Counter2 = () => {
  const [count , setCount] = useState(0);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}

const App = () => {
  return (
    <>
      <Counter1 />
      <Counter2 />
    </>
  )
}
```

위 예제를 살펴봤을때 Counter1과 Counter2 컴포넌트에는 고유한 지역 상태가 있기 때문에 컴포넌트에서 보여주는 숫자는 격리되어 있다. 위와 같이 두 컴포넌트가 하나의 카운트 상태를 공유하게 하려면 Context를 통해서 해결할 수 있다.

```jsx
const CountContext = createContext();

const CountProvider = ({children}) => (
  <CounterContext.Provider value={useState(0)}>
    {children}
  </CounterContext.Provider>
);

const Counter1 = () => {
  const [count , setCount] = useContext(CountContext);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}

const Counter2 = () => {
  const [count , setCount] = useContext(CountContext);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}

const App = () => {
  return (
    <CounterProvider>
      <Counter1 />
      <Counter2 />
    </CounterProvider>
  )
}
```

위와 같은 구조로 만든다면 카운터 상태를 공유할 수 있고 숫자가 동시에 증가한다.

컨텍스트 예제와 Jotai의 차이점을 알아보자.

### 구문 단순성

```jsx
const countAtom = atom(0);

const Counter1 = () => {
  const [count , setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}

const Counter2 = () => {
  const [count , setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return <>{count} <button onClick={inc}>+1</button></>;
}
```

useState 예제와 거의 흡사하게 atom이라는 개념이 추가된 것 이외에 달라진게 없어보인다.
추가적으로 위와 같이 Jotai를 사용하면 Context 사용없이 쓸 수 있다.

```jsx
const App = () => {
  return (
    <>
      <Counter1 />
      <Counter2 />
    </>
  )
}
```

### 동적 아톰 생성

Jotai에 새로운 기능은 동적 아톰 생성 기능이다. Jotai의 스토어는 기본적으로 아톰 구성 객체와 아톰 값으로 구성된 WeakMap 객체이다. 아톰 구성 객체( atom config object )는 atom 함수로 생성된다. Jotai의 구독은 아톰 기반이므로 useAtom 훅이 store에 있는 특정 아톰을 구독한다는 것을 의미한다.

❓ WeakMap이란 무엇인가?

MDN에 따르면 우리가 아는 Map인데 key로는 객체와 symbol 값을 전달받고 사용하지 않으면 GC에 의해서 바로 수거된다고 한다. 

💬 현업 코드내에서 WeakMap을 어떤 케이스에 쓸 수 있을까?? 잘 떠오르지 않는다.

### 렌더링 최적화

아톰은 렌더링을 감지하는 단위이다. 아톰을 원시값처럼 원하는 만큼 작게 만들어서 리렌더링을 제어할 수 있다.

```jsx
const firstNameAtom = atom('React');
const lastNameAtom = atom('Hooks');
const ageAtom = atom(3);
```

위와 같이 선언하고 특정 컴포넌트에서 age와 같은 값을 사용하지 않았다면 age값이 바뀌더라도 해당 컴포넌트는 리렌더링 되지 않는다. 다만 위와 같은 방식은 원시값으로 개별적으로 관리해야함에 따라서 불편한 부분이 많은데 위와 같은 문제를 해결하기 위해서 **파생 아톰**의 개념이 있다.

```jsx
const personAtom = atom((get) => ({
  firstName : get(firstName),
  lastName : get(lastName),
  age : get(age),
}));
```

personAtom의 경우에는 3가지 속성 중 하나가 변경될때마다 갱신된다. 이것을 **의존성 추적** 이라고 표현한다.

```jsx
const PersonComponent = () => {
  const person = useAtom(personAtom);

  return <>{person.firstName} {person.lastName}</>
}
```

그렇지만 이 코드는 예상과는 다르게 동작한다. ageAtom의 값이 변경되면 리렌더링되므로 불필요한 리렌더링이 발생한다.

> ❓ 위에서 3가지 속성 중 하나가 변경될때마다 갱신된다고 했는데 왜 예상과 다르다는거지??
> 

위와 같이 리렌더링을 피하려면 값만 포함하는 파생 아톰을 만들어야 한다.

```jsx
const fullNameAtom = atom((get) => ({
  firstName : get(firstName),
  lastName : get(lastName),
}))
```

> ❗ 파생 아톰을 만들어서 사용해야하는건 하나의 관심사로 이루어진 특정 객체를 분리해서 사용해야한다는건데 이건 너무 불편할 것 같다는 생각이든다.