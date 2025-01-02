## 사용 사례 시나리오 4: React Tracked

`React Tracked`는 속성 감지를 기반으로 자동으로 렌더링 최적화를 수행하는 라이브러리다. 다른 상태 관리 라이브러리와 함께 사용할 수 있다는 특징이 있다.

⚠️ 중요한 점은, 상태 관리 기능을 제공하지 않으면서 렌더링 최적화 기능을 제공하는 라이브러리라는 점이다.

### useState와 함께 React Tracked 사용하기

```tsx
import { createContainer } from "react-tracked";

const { Provider, useTracked } = createContainer(useValue);

const Counter = () => {
  const [state, useState] = useTracked();
  const inc = () => {
    setState((prev) => ({ ...prev, count: prev.count + 1 }));
  };

  return (
    <div>
      count: {state.count}
      <button onClick={inc}>+1</button>
    </div>
  );
};
```

기존에 `useTracked` 대신 `useStateContext`를 사용한 다수의 Counter가 동일한 컨텍스트를 공유한다고 치면, 하나의 Counter의 속성만 업데이트 되어도 다른 모든 Counter가 리렌더링된다는 단점이 있다.

다만 `useTracked`를 사용함으로써 반환된 state 객체는 추적되고, 해당 Counter의 속성이 변경된 경우에만 리렌더링을 감지하게 된다.

### useReducer와 함께 React Tracked 사용하기

```tsx
// "INC"나 "SET_TEXT" 액션 타입에 따라 다른 상태 업데이트를 수행하도록 구성
const useValue = () => {
  type State = { count: number; text: string };
  type Action = { type: "INC" } | { type: "SET_TEXT"; text: string };

  const [state, dispatch] = useReducer(
    (state: State, action: Action) => {
      if (action.type === "INC") {
        return { ...state, count: state.count + 1 };
      }

      if (action.type === "SET_TEXT") {
        return { ...state, test: action.text };
      }

      throw new Error("unknown action type");
    },
    { count: 0, text: "hello" }
  );

  useEffect(() => {
    console.log("latest state", state);
  }, [state]);

  return [state, dispatch] as const;
};
```

```tsx
import { createContainer } from "react-tracked";

const { Provider, useTracked } = createContainer(useValue);

const Counter = () => {
  const [state, dispatch] = useTracked();
  const inc = () => dispatch({ type: "INC" });

  return (
    <div>
      count: {state.count}
      <button onClick={inc}>+1</button>
    </div>
  );
};
```

### 향후 전망

React Tracked 구현은 `proxy-compare`(프록시 객체를 활용한 상태 전후 비교), `use-context-selector`(셀렉터를 활용하여 독립된 컨텍스트 범위 선택 후 리렌더링), 총 두 개의 내부 라이브러리에 의존한다.

향후 리액트에서 useContextSelector 또는 유사한 형태의 구현체가 등장할 가능성이 있어서, React Tracked는 use-context-selector에서 네이티브 useContextSelector로 마이그레이션하는 것이 가능해질 수 있다. 이로 인해서, 리액트와 완벽한 호환성을 제공할 것으로 보여진다.
