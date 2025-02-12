# 10. 사용 사례 시나리오 4: React Tracked

> React Tracked 이해하기 (useState, useReducer, React Redux와 함께 사용하기)

<br/>

## 🔖 1. React Tracked란?

### 상태 사용 추적

- React Tracked는 속성 감지를 기반으로 자동으로 렌더링 최적화를 수행하는 상태 사용 추적 라이브러리다.
- Redux, Zustand 등 다른 상태 관리 라이브러리와 함께 사용할 수 있다.
- 렌더링 최적화 기능을 제공한다는 점에서 단순 전역 상태 관리 라이브러리와는 다르다.

### Context와의 비교

- 특정 값만을 변경했는데 새로운 컨텍스트 값이 전파되고, `useContext`는 리렌더링을 감지한다. 따라서 불필요한 리렌더링이 발생할 수 있다.
- React Tracked를 사용하면 `useContext` 대신 사용할 수 있는 `useTracked` 훅을 정의한다. 이는 상태를 proxy로 감싸고 사용을 추적한다.

### 예시 코드

```jsx
const useFirstName = () => {
  const [{ firstName }] = useTracked();
  return firstName;
};
```

### 예시를 통해 본 특징

- 일단 `useContext`와 사용법이 같기에 적용하기 쉽다는 장점이 있다.
- 코드는 Context를 사용했을 때와 크게 다르지 않으나, 내부에서 상태 사용을 추적하고 렌더링을 자동으로 최적화한다! 매우 기특한 녀석 👍
- 이후 나오는 `useState`와 `useReducer`와 함께 사용하는 방법에도 주로 위와 같은 내용을 다루고 있다.

<br/>

## 🔖 2. `useTracked`의 장단점

### `useTracked` 특징과 장점

- 위에서 언급했듯이 Proxy 기반으로 추적한다. `useTracked`은 상태 객체를 Proxy로 감싸거나, 그와 유사한 기법을 사용하여 속성(get) 접근을 가로챈다. (비슷한 내용이 9장: valtio 때도 나왔다) 그 결과, 컴포넌트가 해당 상태 객체의 어떤 속성을 실제로 읽었는지(접근했는지)를 기록할 수 있다.

- 접근된 속성만 리렌더링을 트리거 한다. 렌더링이 끝나면 `useTracked`은 "이 컴포넌트는 A, B, C 속성에 의존하고 있네" 라고 인지하고 해당 속성들만 추적한다. 이후 A, B, C 중 하나라도 값이 변하면 해당 훅을 사용하는 컴포넌트를 리렌더링한다. 반면, 사용하지 않은 속성은 바뀌어도 이 컴포넌트를 다시 그리지 않는다. (아주 효율적이야)

- 리렌더링 최적화를 한다. 전역 상태나 큰 객체에서 정말로 필요한 필드만 “실제 의존”으로 간주하기 때문에, 불필요한 렌더를 크게 줄일 수 있다. 예를 들어 `user.name`만 접근했다면 `user.age`나 `user.email`이 변해도 컴포넌트가 재렌더링되지 않는다.

### `useTracked` 단점

- 각 렌더링 과정에서 어떤 속성을 읽었는지 기록하고, Proxy를 통해 속성 접근을 감지하는 과정에서 성능 overhead가 발생할 수 있을 것 같다. 일반적인 상황에서는 큰 문제가 안 되겠지만, 복잡한 데이터 구조나 빈번한 렌더링이 있는 경우 주의하면 좋겠다.

- 상태가 깊게 중첩된 구조일수록 추적 로직이 많아지기 마련이다. 예를 들어, `obj.a.b.c`처럼 깊숙이 들어가는 속성까지 추적해야 하는 경우, 생각치 못한 성능 비용이나 예상치 못한 재렌더링이 발생할 수 있을듯 하다. (하지만 이는 비단 `useTracked`의 단점만은 아닌 것 같다)

- 평소 Redux 같이 명시적 액션/리듀서 구조에 익숙한 팀이라 `useTracked`가 제공하는 자동 추적은 추상적이라고 느낄 수 있을듯 하다. 상태 변경 흐름이 명확하게 보이지 않기에 상태 추적 / 디버깅이 어려울 수도 있겠다.

<br/>

## 📚 References

- [dai-shi/react-tracked](https://github.com/dai-shi/react-tracked)
