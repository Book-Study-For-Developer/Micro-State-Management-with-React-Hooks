## 전역 상태 관리 문제 해결하기

### 전역 상태를 설계할 떄의 문제점

1. **전역 상태를 읽는 방법**

   전역 상태는 여러 개의 값을 가질 수 있지만, 컴포넌트는 일부 값만 필요할 수 있다. 전역 상태가 바뀜에 따라 상관없는 값에 의한 리랜더링이 일어나느 것은 바람직하지 않기 때문에 리랜더링 최적화가 필요하다.

2. **전역 상태에 값을 넣거나 갱신하는 방법**

   전역 상태는 여러 값을 가질 수 있고, 그 중에는 중첩된 객체를 가질 수 있다. 이때 전역 변수의 일부를 가지고 직접 변경하는 것은 좋은 방법이 아니다.

→ 전역 상태 변경을 감지하기 위해 전역 상태를 변경하는 함수를 따로 제공해야하고, 일부는 상태를 직접 변경할 수 없도록 클로저를 활용하여 숨기는 경우도 있다.

#### 전역 상태 관리 vs 범용 상태 관리

전역 상태 관리

- 애플리케이션의 전체 컴포넌트 트리에서 공유되는 상태를 관리하는 것
- 이 상태는 모든 컴포넌트가 접근할 수 있으며, 전역적으로 데이터를 읽거나 수정할 수 있음

범용 상태 관리

- 애플리케이션에서 모든 상황에 적합하도록 설계된 상태 관리 방식
- 단순히 전역 상태뿐만 아니라, 전역이 아닌 컴포넌트 간 상태, 하위 트리 간 상태를 포함함

<br />

## 데이터 중심 접근 방식과 컴포넌트 중심 접근 방식

**데이터 중심 접근 방식**

데이터 중심 접근 방식은 데이터가 리액트 외부 자바스크립트 메모리에 있기 때문에 모듈 상태를 사용하는 것이 적합하다. 리액트 컴포넌트가 랜더링 되기 전이나, 언마운트 되었을 때도 이 컴포넌트가 존재한다.

데이터 중심 접근 방식을 사용하는 전역 상태 관리 라이브러리는 외부 상태에 접근하여 리액트 컴포넌트가 갱신가능하도록 store 객체로 감싸 관리하고, 리액트 컴포넌트에 연결하는 API를 제공한다.

**컴포넌트 중심 접근 방식**

컴포넌트 중심 접근 방식은 컴포넌트 내부에 필요한 데이터의 Props Drilling을 개선하기 위해 전역에서 관리하기 때문에 데이터 모델이 컴포넌트에 높은 의존성을 가진다.

컴포넌트 중심 접근 방식은 생명 주기 내에서 전역 상태를 유지하고, 언마운트 되었을 떄 상태도 함께 사라져야한다. 서로 다른 컴포넌트 하위 트리에 존재하는 동일한 전역 상태가 존재할 수 도 있다.

<br />

## 리랜더링 최적화 패턴

중첩된 객체를 활용할 수 있는 상태에서 리랜더링을 최적화는 것은 어느 부분이 사용되는지 지정하고, 비교하는 것이다. 아래 3가지 방식은 객체 내부의 일부분에 접근하는 방식(개념)을 소개하고 있다.

**1. 선택자 함수**

선택자 함수는 상태를 받아 상태의 일부를 반환한다.

```tsx
const Component = () => {
  const value = useSelector((state) => state.b.c);

  return <>{value}</>;
};
```

컴포넌트 내부에서 `state.b.c` 만 사용하기 때문에 다른 값의 변경에 의한 리랜더링을 방지해야한다.

useSelector를 사용하면 상태가 변경될 때마다 선택자 함수의 결과를 비교하는데 사용할 수 있다.

따라서 동일한 입력에 대해서 동일한 결과를 반환하는 것이 중요하기 때문에 메모이제이션을 활용한다.

**2. 속성 접근 감지**

속성 접근을 감지하고 감지한 정보를 바탕으로 랜더링 최적화에 활용할 수 있는 상태 사용 추적을 적용하면 랜더링 최적화가 가능하다. → useTrackedState()

useSelector로 이러한 기능을 만들기 위해서는 메모이제이션이나 사용자 지정 비교 함수와 같은 복잡한 기법이 필요하다.

useTrackedState()를 구현하기 위해 객체에 대한 속성 정보를 확인할 수 있는 Proxy가 필요하다. 이를 활용하여 useTrackedState()라는 훅을 만들면 자동으로 랜더링 최적화가 수행될 수 있다.

**3. 아톰 사용**

아톰(Atom)은 리랜더링을 발생시키는 최소 상태 단위를 의미한다.

객체 내에 값을 별도의 아톰으로 구분한다면 각 아톰이 별도의 전역 상태를 갖는 것 처럼 활용할 수 있다.

```tsx
const globalState = {
  a: atom(1),
  b: atom(2),
  c: atom(3),
};

const Component = () => {
  const value = useAtom(globalState.a);

  return <>{value}</>;
};
```

<br />

## **🧐 Proxy in JS**

Proxy는 Javascript 내장 객체로서 기본 작업을 가로채서 재정의할 수 있다.

```tsx
const proxy = new Proxy(target, handler);
// target : 원본 객체
// handler : 가로채는 작업과 재정의 방법을 정의하는 객체
```

```tsx
const handler = {
  get(obj, prop) {
    return prop in obj ? obj[prop] : 37;
  },
};

const p = new Proxy({}, handler);
p.a = 1;
p.b = undefined;

console.log(p.a, p.b);
//  1, undefined

console.log('c' in p, p.c);
//  false, 37
```

Reference : [https://inpa.tistory.com/entry/JS-📚-자바스크립트-Proxy-Reflect-고급-기법#javascript*reflect*객체](https://inpa.tistory.com/entry/JS-%F0%9F%93%9A-%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-Proxy-Reflect-%EA%B3%A0%EA%B8%89-%EA%B8%B0%EB%B2%95#javascript_reflect_%EA%B0%9D%EC%B2%B4)

책에서 속성 접근을 확인하기 위해 Proxy를 사용한다고 했는데 아래와 같이 속성에 접근했을 때 원하는 로직을 실행할 수 있을 것 같다.

```jsx
const user = {
  name: 'Park',
  age: 25,
};

const loggingProxy = new Proxy(user, {
  get(target, key) {
    // 접근했을 때 처리할 로직
    console.log(`Accessing property "${key}": ${target[key]}`);
    return target[key];
  },
});

console.log(loggingProxy.name); // Accessing property "name": Park
```

→ JS Proxy 객체가 활용되는 방식을 조사해보았을 때 api 요청을 래핑해서 네트워크 요청을 가로채거나 로그를 남기는데 활용할 수 도 있다는 것을 확인했고, 이 동작이 axios.interceptor와 유사하다는 느낌을 들었다.

interceptor가 new Proxy()를 내부적으로 호출하고 있는지 궁금했다.

axios.interceptor 내부적으로는 `~InterceptorChain` 로 정의된 비동기 함수 배열을 promise chaining으로 실행시키고 있다고 하고, 앞선 예시처럼 `new Proxy()`를 직접 활용하고 있지는 않았다.

https://github.com/axios/axios/blob/v1.x/lib/core/Axios.js#L124

Reference : https://ko.javascript.info/promise-chaining

<br />
