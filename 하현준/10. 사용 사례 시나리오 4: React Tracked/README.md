## React Tracked 알아보기

> **React Tracked is a library to provide so-called "state usage tracking." It's a technique to track property access of a state object, and only triggers re-renders if the accessed property is changed. Technically, it uses Proxies underneath, and it works not only for the root level of the object but also for deep nested objects.**

React Tracked는 소위 "상태 사용 추적"을 제공하는 라이브러리입니다. 상태 개체의 속성 액세스를 추적하고 액세스된 속성이 변경된 경우에만 다시 렌더링을 트리거하는 기술입니다. 기술적으로는 아래에 있는 프록시를 사용하며 개체의 루트 수준뿐만 아니라 깊게 중첩된 개체에도 작동합니다

>

```tsx
const NameContext = creatContext([
	{ firstName: 'react', lastName: 'hooks',
	() => {},
]);
coonst useFirstName = () => {
	const [{ firstName }] = useContext(NameContext);
	return firstName;
}
```

사용하는 입장에서는 firstName만 사용했지만 lastName을 변경하더라도 리렌더링이 일어나기 때문에 바람직하지 않은 동작 방식이다.

이럴 때 React Tracked 라이브러리에서 제공해주는 useTracked 훅으로 사용하면 위와 같은 상황을 해결해 준다.

### useState, useReducer와 함께 React Tracked 사용하기

라이브러리 공식문서에 useState의 똑같은 예시가 존재한다.

```tsx
import { useState } from "react";

const useValue = () =>
  useState({
    count: 0,
    text: "hello",
  });

const { Provider, useTracked } = createContainer(useValue);

// 해당 상태를 가져와 사용
const Counter = () => {
  const [state, setState] = useTracked();
  const increment = () => {
    setState((prev) => ({
      ...prev,
      count: prev.count + 1,
    }));
  };
  return (
    <div>
      <span>Count: {state.count}</span>
      <button type="button" onClick={increment}>
        +1
      </button>
    </div>
  );
};
```

❓여러 상태를 사용할 수 있으니 함수명은 바꿔서 사용해야 할 듯 하다.

```tsx
const { Provider, useTracked: useCountStateTracked } =
  createContainer(useValue);

const [state, setState] = useCountStateTracked();
```

useReducer를 사용하는 예시도 기존 useReducer사용법과 동일하지만 라이브러리를 쓰는 것 외에 특별한 것은없다.

https://github.com/wikibook/msmrh/blob/main/chapter10/03_with_usereducer/src/App.tsx

해당 라이브러리에서 필요한 상태만 추적해 리렌더링이 가능한 이유는 [`use-context-selector`](https://www.npmjs.com/package/use-context-selector)라는 라이브러리를 내부적으로 사용하기 때문이다.

❓ 내부 라이브러리를 살펴본 결과 `use-context-selector` 라이브러리를 편하게 사용할 수 있게 헬퍼 함수를 제공해주는 느낌이 강했다. context도 만들어주고, provider도 제공해주는 역할을 하는 라이브러리?

https://github.com/dai-shi/react-tracked/blob/main/src/createContainer.ts

라이브러리 하단에 이런 이슈가 쓰여있다.

> This library internally uses `use-context-selector`, a userland solution for `useContextSelector` hook. React 18 changes useReducer behavior which `use-context-selector` depends on. This may cause an unexpected behavior for developers. If you see more `console.log` logs than expected, you may want to try putting `console.log` in useEffect. If that shows logs as expected, it's an expected behavior. For more information:

재랜더렝이 동작하는 것처럼 보이지만 useEffect내부에 콘솔을 찍으면 한번만 동작할 것이고, 이는 리엑트18의 동시성 렌더링이 업데이트 되면서 발생하는 이슈라고 판단하고 있다.

>

책에서도 마찬가지로 추후에 리엑트에서 useContextSelector를 만들 것이라고 기대하고 있으며, 라이브러리를 사용중이라면 쉽게 마이그레이션이 가능할 것이라고 제안하고 있다.
