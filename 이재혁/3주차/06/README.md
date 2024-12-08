# 06. 전역 상태 관리 라이브러리 소개

## 전역 상태 관리 문제 해결하기

- 전역 상태 설계시 발생하는 두 가지 문제점
  1. 전역 상태를 읽는 방법
     - 전역 상태를 사용하지 않는 컴포넌트도 리렌더링이 발생 -> 최적화 필요
  2. 전역 상태에 값을 넣거나 갱신하는 방법
     - 객체를 예로 들면 직접 값을 변경하는 것은 리액트에서 캐치를 못할 가능성도 있고 불변성을 지키지 못해 예기치 못한 상황이 발생할 수 도 있음
     - 이를 위해 리액트에서 사용하는 방식으로 getter와 setter 함수가 필요

---

### 전역 상태 관리 vs 범용 상태 관리

범용 상태 관리

- 범용 상태 관리는 상태의 위치(전역/지역)에 관계없이 일관된 패턴으로 상태를 관리하는 방식
- Flux 패턴을 활용한 단방향 데이트 흐름의 방식을 사용하는 Redux
- 상태머신 기반의 방식인 XState
- 범용 상태 관리 방식은 전역뿐 아니라 지역 상태에서도 유용하다
- 그러면 이 둘의 차이를 알아보고 상태 머신에 대해 알아보자

#### 상태머신(State Machine)이란?

- 상태머신(State Machine)

  - [유한 상태 머신(Finite State Machine, FSM)](https://ko.wikipedia.org/wiki/%EC%9C%A0%ED%95%9C_%EC%83%81%ED%83%9C_%EA%B8%B0%EA%B3%84)에서 파생된 개념
    - 설명을 몇 줄 읽자마자 창을 닫아버렸다.
    - 유한, 추상, 하나의 상태, 현재 상태, 사건(event)이라는 트리거, 이를 통해 상태는 변함 이를 전이(transition)
  - 유한한 상태 집합과 이들 사이의 전환(transition)을 명시적으로 정의한 모델
  - 상태머신은 다음과 같은 구조를 가짐
    1. 상태(states): 시스템이 가질 수 있는 모든 가능한 상태들.
    2. 전환(transitions): 한 상태에서 다른 상태로 전환되는 규칙.
    3. 이벤트(events): 전환을 트리거하는 외부 또는 내부 이벤트.
    4. 동작(actions): 상태가 전환되거나 유지될 때 실행되는 특정 동작.

- `XState`는 이러한 상태와 전환을 코드로 모델링하여 어플리케이션의 복잡한 상태 로직을 쉽게 다룰 수 있도록 돕는 라이브러리

1. 만약에 간단한 `toggle` 기능이 있다고 가정했을 때 다른 라이브러리는 추후 배우게 될 테니 원활한 비교를 위해 `useState`를 사용한 예제와 비교

1-1. `useState` 를 사용하는 경우

```tsx
// useState
import React, { useState } from "react";

function App() {
  // 상태 관리
  const [state, setState] = useState("inactive"); // 초기 상태: "inactive"

  // 상태 전환 로직
  const toggle = () => {
    setState((prev) => (prev === "inactive" ? "active" : "inactive"));
  };

  return (
    <div>
      <p>현재 상태: {state}</p> {/* 현재 상태 출력 */}
      <button onClick={toggle}>Toggle</button> {/* 상태 전환 */}
    </div>
  );
}

export default App;
```

1-2. `XState`를 사용한 경우

- 간단한 머신 설명
  - 상태: `닫힘`과 `열림`.
  - 이벤트: `열기 버튼 클릭`, `닫기 버튼 클릭`.
  - 전환: `닫힘` → `열림` (열기 버튼 클릭 시) 또는 `열림` → `닫힘` (닫기 버튼 클릭 시).
- `createMachine`는 core 로직임
  - subscribe를 통해 상태 변화 트래킹이 가능
  - start 메서드를 통해 머신 가능
  - send를 통해 이벤트 발생

```tsx
// 상태 머신
import { createMachine, interpret } from "xstate";
import { useMachine } from "@xstate/react";

// 상태머신 정의
const toggleMachine = createMachine({
  id: "toggle",
  initial: "inactive",
  states: {
    inactive: {
      on: { TOGGLE: "active" }, // TOGGLE 이벤트에 따라 active로 전환
    },
    active: {
      on: { TOGGLE: "inactive" }, // TOGGLE 이벤트에 따라 inactive로 전환
    },
  },
});

// React 컴포넌트에서 사용
function App() {
  // XState의 useMachine 훅 사용
  const [state, send] = useMachine(toggleMachine);

  return (
    <div>
      <p>현재 상태: {state.value}</p> {/* 현재 상태 출력 */}
      <button onClick={() => send("TOGGLE")}>Toggle</button> {/* TOGGLE 이벤트 전송 */}
    </div>
  );
}

export default App;
```

- 플로우
  - `createMachine`:
    - 상태를 모델링하고 상태 간 전환 규칙을 정의합니다.
    - `initial`: 초기 상태 (`inactive`).
    - `states`: 각 상태와 상태에서 처리할 이벤트 정의.
  - `useMachine`:
    - React 컴포넌트에서 XState 상태 머신을 사용하기 위한 훅.
    - `state`: 현재 상태 값 (`inactive` 또는 `active`).
    - `send`: 상태 전환을 트리거하는 함수 (`send("TOGGLE")`).
  - 상태 전환:
    - `send("TOGGLE")` 호출 시 현재 상태에 따라 정의된 전환 로직에 따라 상태가 변경됩니다.

---

## 데이터 중심 접근 방식과 컴포넌트 중심 접근 방식 사용하기

### 1. 데이터 중심 접근 방식 이해하기

- 이미 컴포넌트 외부에 상태가 정의되어 있음
- 싱글톤 패턴
- 컴포넌트가 마운트되거나 해제된 이후에도 상태는 남을 수 있음
- 모듈 상태를 생성하고 컴포넌트와 연결짓는 방법을 제공함

### 2. 컴포넌트 중심 접근 방식 이해하기

- 데이터 모델이 컴포넌트에 강한 의존성을 가지고 있다.
- 팩토리 패턴
- 컴포넌트에서 쓸 수 있는 전역 상태를 초기화할 수 있는 메서드를 제공 -> 컴포넌트에서 실행해서 리액트 생명 주기 내에서 관리하도록 함
- 팩토리 함수가 직접적으로 전역 상태를 만들어 내지는 않음 -> 컴포넌트를 거쳐야함

---

## 리렌더링 최적화

- 최적화의 핵심 ! -> state의 특정 부분을 선택하는 것
- 이를 위한 방식 세 가지가 있다.

### 1. 선택자 함수 사용

- 중첨 객체가 있을 때
- 특정 객체의 값에 접근하는 selector를 만듬
- 컴포넌트가 사용할 값을 명시적으로 지정해주는 것
- 참조형이라서 같은 객체라고 판별할 수 있도록 deepEqual로 비교해주어야함
- 수동 최적화임

```tsx
const Component = () => {
  const value = useSelector((state) => state.b.c); // state.b.c * 2 처럼 파생된 값 반환도 가능ㄴ
  return <div>{value}</div>;
};
```

### 2. 속성 접근 감지

- 선택자 함수 처럼 명시적으로 지정하는 방법외에도 있다.
- 속성 접근을 감지하고 감지된 장보를 렌더링 최적화에 사용할 수 있는 상태 사용 추적이 있다.

e.g.) 상태 사용 추척 기능이 있는 useTrackedState 훅이 있다.

- 프록시를 이용한 구현
- [프록시](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

  - Proxy 객체를 사용하면 원래 Object 대신 사용할 수 있는 객체를 만듬
  - 이 객체의 속성 가져오기, 설정 및 정의와 같은 기본 객체 작업을 재정의 가능
  - 프록시 객체는 일반적으로 속성 액세스를 기록하고, 입력의 유효성을 검사하고, 형식을 지정하거나, 삭제하는 데 사용됨

- 두 개의 매개변수를 사용하여 Proxy를 생성
  - target: 프록시할 원본 객체
  - [handler](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy): 가로채는 작업과 가로채는 작업을 재정의하는 방법을 정의하는 객체

```tsx
// GPT에게 시켜본 useTrackedState 만들기
function useTrackedState(initialState: any) {
  const [_, forceUpdate] = useState(0); // 강제 렌더링을 위한 state
  const stateRef = useRef(initialState); // 실제 상태를 저장
  const trackedPaths = useRef(new Set()); // 읽힌 경로를 추적

  // 프록시 생성
  const proxy = new Proxy(stateRef.current, {
    get(target, prop) {
      trackedPaths.current.add(prop); // 읽힌 경로 기록
      return target[prop];
    },
    set(target, prop, value) {
      if (target[prop] !== value) {
        target[prop] = value; // 값 업데이트
        forceUpdate((v) => v + 1); // 변경된 경우 강제 렌더링
      }
      return true;
    },
  });

  useEffect(() => {
    trackedPaths.current.clear(); // 렌더링 후 읽힌 경로 초기화
  });

  return proxy;
}
```

```tsx
const Component = () => {
  const trackedState = useTrackedState();
  return <div>{trackedState.b.c}</div>;
};
```

- useSelector vs useTrackedState
  - 값에 따른 조건부 렌더링이 구현되어있을 때
  - useSelector의 경우 조건이 바뀔 때만 리렌더링
  - useTrackedState의 경우 값이 바뀔 때마다 리렌더링

### 3. 아톰 사용

- 아톰 - 리렌더링을 발생시키는 데 사용되는 최소 상태 단위
- 전체 전역 상태를 구독하던 방식을 좀 더 세분화해서 구독하는 것

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

- jotai가 해당 패턴을 사용
- 수동 최적화와 자동 최적화의 중간 단계
- 값의 정의는 명시적
- 의존성 추적은 자동
