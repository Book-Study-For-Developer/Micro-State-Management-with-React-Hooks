# 09. 사용 사례 시나리오 3: Valtio

> proxy를 사용한 변경 감지와 불변 상태 생성, 리렌더링 최적화

<br/>

## 🔖 1. Valtio란?

### 특장점

- Proxy를 사용하여 상태를 추적하고, 상태 변경 시 관련 컴포넌트만 자동으로 업데이트 해주는 방식
- 즉, 상태 객체를 직접 다루듯이 편하게 수정하면, Valtio가 그 변화를 감지하고 필요한 부분만 리렌더링해 준다.
- 상태 객체를 Proxy로 감싸 상태에 대한 모든 접근을 감지한다. 어떤 컴포넌트가 상태의 어느 부분을 읽었는지 파악하고, 그 부분이 변할 때만 그 컴포넌트를 다시 렌더링한다.
- `useSnapshot` 훅으로 현재 상태 스냅샷을 읽을 수 있으며, 상태 변경 시 자동으로 업데이트된 스냅샷을 받아볼 수 있다.
- 상태를 기존 JS 객체 다루듯이 변경하면 되므로 러닝 커브가 낮다.
- Proxy 기반이라 예측하기 힘든 시나리오에선 주의가 필요할 수 있다.

### 예시 코드

```jsx
import { proxy, useSnapshot } from 'valtio';

// 1. 상태 정의: Proxy로 상태를 감싸는 proxy 함수를 사용
const state = proxy({
  count: 0,
  text: 'Hello',
});

// 2. 상태를 읽고 변경하는 컴포넌트
function Counter() {
  // useSnapshot으로 현재 상태 스냅샷 얻기
  const snap = useSnapshot(state);
  return (
    <div>
      <p>Count: {snap.count}</p>
      <button onClick={() => state.count++}>+1</button>
    </div>
  );
}

function TextDisplay() {
  const snap = useSnapshot(state);
  return (
    <div>
      <p>Text: {snap.text}</p>
      <button onClick={() => (state.text = state.text + '!')}>Add "!"</button>
    </div>
  );
}

export default function App() {
  return (
    <div>
      <h1>Valtio Example</h1>
      <Counter />
      <TextDisplay />
    </div>
  );
}
```

<br/>

## 🔖 2. 언제 Valtio를 사용하면 좋을까?

### 언제 Valtio를 사용하면 좋을까?

- 전역 상태를 쉽게 정의하고 갱신하고 싶을 때
- 상태 변경을 `state.count++`와 같이 직관적으로 하고 싶을 때

### 언제 Valtio를 사용하지 않는 것이 좋을까?

- 대규모 프로젝트에서 엄격한 패턴이 필요할 때(이럴 때는 Redux의 action, reducer 처럼 일관된 패턴을 쓰는 것이 좋을듯)
- Redux의 MobX처럼 복잡한 디버깅이나 추적 기능이 필요한 경우, Valtio의 Proxy 기반 렌더링 최적화가 단점이 될 수 있다.
- 타입 안정성이 필요할 때는 Valtio의 느슨한 구조가 단점이 될 수 있다.
