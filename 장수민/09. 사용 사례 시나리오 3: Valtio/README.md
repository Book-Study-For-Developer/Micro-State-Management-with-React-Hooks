## 09. 사용 사례 시나리오 3: Valtio

Valtio는 Zustand나 Jotai와는 다르게 변경 가능한 갱신 모델(mutating update model)을 기반으로 하는 또 다른 전역 상태 관리 라이브러리다.

리액트와의 통합을 위해 Valtio는 프락시를 사용해 변경 불가능한 스냅숏을 가져온다.

또 다른 특징으로는,

- API는 자바스크립트만으로 이뤄져있고 모든 작업이 내부에서 처리된다.
- Valtio는 프락시를 활용해서 자동으로 리렌더링을 최적화한다.
- 리렌더링을 제어하기 위해 선택자가 필요하지 않다.
- Valtio의 자동 렌더링 최적화는 상태 사용 추적(stage usage tracking)이라는 기법을 기반으로 한다.

> [!NOTE]
> 상태 사용 추적을 사용하면 상태의 어느 부분이 사용되는지 감지할 수 있고, 사용된 부분이 변경될 경우에만 컴포넌트를 리렌더링되게 할 수 있다. 결과적으로는 개발자가 작성해야 할 코드가 줄어든다.

### Valtio의 원리

```tsx
const proxyObject = new Proxy(
  {
    count: 0,
    text: "hello",
  },
  {
    set: (target, prop, value) => {
      console.log("start setting", prop);
      target[prop] = value;
      console.log("end setting", prop);
    },
  }
);
```

첫 번째 인수로 객체를 넣고, 핸들러를 담는 컬렉션 객체를 두 번째 인수를 넣어 Proxy 객체를 생성한다. set 핸들러를 담았으므로, 객체의 값이 갱신되려고 할 때 실행된다.

### 프락시를 활용한 변경 감지 및 불변 상태 생성하기

Valtio는 프락시를 사용해서 변경 가능한 객체에서 변경 불가능한 객체를 생성한다. 이 불변 객체를 스냅숏이라고 한다.

```tsx
import { proxy, snapshot } from "valtio";

// proxy에서 반환하는 state 객체는 변경을 감지하는 프락시 객체
const state = proxy({ count: 0 });

// Object.freeze로 동결되어 변경 불가능한 불변 객체
const snap1 = snapshot(state);
```

proxy와 snapshot 함수는 중첩된 객체에 대해서도 작동하고 스냅숏 생성을 최적화한다. **즉, snapshot 함수는 속성이 변경될 때만 새 스냅숏을 생성한다.**

### 프락시를 활용한 리렌더링 최적화

```tsx
// proxy와 useSnapshot은 Valtio에서 제공하는 두 가지 주요 기능이고, 대부분의 경우에 사용된다.
import { proxy, useSnapshot } from "valtio";

const state = proxy({
  count1: 0,
  count2: 0,
});

const Counter1 = () => {
  // useSnapshot에서 반환하는 값의 변수명을 snap으로 설정하는 것이 관례다.
  // 접근은 useSnapshot 훅에 의해 추적 정보로 감지되며, 필요한 경우에만 리렌더링한다.
  const snap = useSnapshot(state);
  const inc = () => ++state.count1;
  return (
    <>
      {snap.count1} <button onClick={inc}>+1</button>
    </>
  );
};
```

기본적으로 일반적인 객체와 배열을 포함하는 어떤 객체도 완전하게 지원되는 데다가, 심지어 깊게 중첩된 경우에도 문제가 없다.

### 프록시 객체를 사용한 접근 방식의 장단점

Valtio는 언제 사용해야 하고 언제 사용하지 말아야 할지를 결정해야 할 필요가 있다.

Valtio는 두 가지 상태 업데이트 모델이 있다. 하나는 불변 갱신이고 하나는 변경 가능한 갱신이다. 자바스크립트 자체는 Object.freeze를 사용하지 않는 한 변경 가능한 갱신을 허용하지만, 리액트는 불변 상태를 중심으로 만들어졌다.

🚨 유의해야 할 점 1. 그래서, 두 모델을 같이 사용하는 경우 혼동하지 않도록 해야 한다. **괜찮은 해결책은 멘탈 모델 전환을 쉽게 할 수 있도록 Valtio의 상태와 리액트의 상태를 명확하게 분리하는 것이다.**

🚨 유의해야 할 점 2. **프락시는 렌더링 최적화를 내부적으로 처리하기 때문에 동작을 디버깅하기가 더 어려울 수 있다는 점을 고려해야 한다.**

반대로, 변경 가능한 갱신의 가장 큰 장점은 네이티브 자바스크립트 함수를 사용할 수 있다는 것이다. 변경 가능한 갱신으로 어플리케이션 코드를 줄이는 데 도움이 된다.
