## 01. 리액트 훅을 이용한 마이크로 상태 관리

### 📚 요약 정리 

- **마이크로 상태 관리**는 좀 더 목적 지향적이며 특정한 코딩 패턴과 함께 사용한다. 
- 리액트에서의 상태는 사용자 인터페이스(UI)를 나타내는 모든 데이터를 칭한다. 
- 마이크로 상태관리를 충족하기 위해서는 상태 읽기 , 갱신 , 상태 기반 렌더링이 기본적으로 필요하다.
- 사용자 정의 훅으로 분리했을때는 컴포넌트를 수정할 필요 없이 로직을 추가할 수 있다는 장점이 있다.
- `Suspense`는 기본적으로 비동기 처리에 대한 걱정 없이 컴포넌트를 코딩할 수 있다. 
- `동시성 렌더링`은 렌더링 프로세스를 청크라는 단위로 분할해서 CPU가 장시간 차단되는것을 방지하는 방법이다. 
- `베일아웃 (bailout)`은 리액트 기술 용어로 리렌더링을 발생시키지 않는 것을 의미한다. 
- `지연 초기화 (lazy initialization)`는 useState가 호출되기 전까지 init 함수는 평가되지 않고 느리게 평가된다. 즉, 컴포넌트가 마운트될때 한번 호출된다. 

### 📝 예제 정리 

```jsx
const Component = () => {
  const [count , setCount] = useState(0);

  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>Set Count to {count + 1}</button>
    </div>
  )
};

```
빠르게 두번 클릭했을때 한번만 증가한다. ==> 이걸 Batching이라고 표현해도 되는건가?!?

**useReducer로 useState를 구현**
```jsx
const useState = (initialState) => {
  const [state , dispatch] = useReducer(
    (prev , action) => 
      typeof action === 'function' ? action(prev) : action, initialState
  );

  return [state , dispatch]
} 
```
애석하게도 useState 내부가 useReducer로 이루어져있다는걸 책을 통해 알게 되었다,,

**useState로 useReducer 구현**

```jsx
const useReducer = () => {
  const [state , setState] = useState(initialState);
  const dispatch = (action) => setState(prev => reducer(prev, action));

  return [state , dispatch];
}
```

### 💡 알아보기

#### 1️⃣ useState와 useReducer의 차이점과 활용법
옛날에 아래 글을 읽고 useReducer를 활용해본 경험이 있는데 복잡한 상태와 검증이 필요할때 useState 대신 useReducer를 활용해보면 좋을 것 같다. 추가적으로 공식문서에서도 useState와 useReducer를 비교한 글이 있다.

🔗 링크
[useState 지옥에서 벗어나기](https://velog.io/@eunbinn/a-cure-for-react-useState-hell)
[useState와 useReducer 비교하기](https://ko.react.dev/learn/extracting-state-logic-into-a-reducer#comparing-usestate-and-usereducer)


#### 2️⃣ Suspense란 무엇인가?

Suspense는 기본적으로 리액트 패러다임에 걸맞는 선언적인 컴포넌트이다. 만일 Suspense를 사용하지 않고 로딩을 처리한다면 어떻게 될까? 







#### 3️⃣ 동시성 렌더링이란 무엇일까?





### ❓ 궁금한 점
- 전역상태를 보통 어떤 상황에서 사용할까? 또는 사용하는 각자의 기준이 있을까?


