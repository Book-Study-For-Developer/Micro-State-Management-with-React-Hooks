## 리액트 훅을 이용한 마이크로 상태 관리

### 마이크로 상태 관리 이해하기

- 리액트에서의 상태 → UI를 나타내는 모든 데이터
- 상태 관리는 리액트 개발 시 중요한 문제 중 하나
- 리액트 훅이 등장 이후: 중앙 집중적인 방식에서 → 경량화, 마이크로화 방식으로
- 전통적인 중앙 집중형 상태 관리가 범용적으로 사용된다면, 마이크로 상태 관리는 좀 더 목적 지향적
- 목적 지향적으로 상태를 관리한다? 특정 목적에 따라 다른 해결책 제공이 가능하다는 의미
  - 폼 상태, 서버 캐시 상태, 내비게이션 상태 등
- 목적 지향적인 방법으로 처리할 수 없는 상태는? 여전히 범용적인 상태 관리가 필요

> 따라서 범용적인 상태 관리를 위한 방법은 가벼워야 하며, 개발자는 요구사항에 따라 적절한 방법을 선택할 수 있어야 한다.

- 마이크로 상태 관리가 가져야 할 기능
  - (필수) 기본적인 상태 관리 기능 - 상태 **읽기** / 상태 **갱신** / 상태 기반 **렌더링**
  - (추가) 리렌더링 최적화 / 호환성 / 비동기 지원 / 파생 상태 / 간단한 문법 등

### 리액트 훅 사용하기

> 리액트 훅은 UI 컴포넌트에서 로직만 추출이 가능 → 커스텀 훅 → 가독성, 테스트, 유지보수 측면에서 이점

- `useState`: 지역 상태를 생성하는 기본적인 함수로 로직을 캡슐화하고 재사용 가능
- `useReducer`: 마찬가지로 지역 상태를 생성할 수 있으며, 복잡한 상태에 있어 `useState`를 대체하는 용도로 자주 사용
- `useEffect`: 리액트 렌더링 프로세스 바깥에서 로직을 실행 가능하며, 리액트 컴포넌트 생명 주기와 함께 동작

### 전역 상태 탐구하기

- 지역상태랑 반대되는 개념으로 애플리케이션 내 서로 멀리 떨어져 있는 여러 컴포넌트에서 사용하는 상태
- 꼭 싱글턴일 필요는 없으며, 싱글턴이 아니라는 의미에서 공유 상태(shared state)라고도 함
- 리액트에서 전역 상태를 구현하는 것이 간단한 작업이 아닌 이유
  - 리액트가 컴포넌트 모델에 기반하기 때문
  - 컴포넌트 모델에서는 지역성이 중요하며, 이는 컴포넌트가 서로 격리돼야 하고 재사용이 가능해야 한다는 것을 의미
- 왜 이렇게 많은 전역 상태 라이브러리가 있나요?
  - 리액트는 전역 상태에 대한 직접적인 해결책을 제공하지 않기 때문에 이는 곧 개발자와 컴퓨니티의 몫으로 남게 됨

### `useState`와 `useReducer`

#### 유사점

- useState

```tsx
const [state, setState] = useState(initialState)
```

- useReducer

```tsx
const [state, dispatch] = useReducer(reducer, initialArg, init?)
```

- **값/함수로 갱신** 모두 가능
- **베일아웃**을 통해 불필요한 리렌더링 방지
- **지연 초기화**를 통해 무거운 계싼이 포함된 초기 상태를 느리게 평가할 수 있음

- `useReducer`로 `useState` 구현하기

```tsx
const reducer = (prev, action) => (typeof action === 'function' ? action(prev) : action)

const useState = initialState => useReducer(reducer, initialState)
```

- `useState`로 `useReducer` 구현하기

```tsx
const useReducer = (reducer, initialArg, init) => {
  const [state, setState] = useState(init ? () => init(initialArg) : initialArg)

  const dispatch = useCallback(action => setState(prev => reducer(prev, action)), [reducer])

  return [state, dispatch]
}
```

#### 차이점

- 초기화 함수 사용하기
  - `useReducer`에서만 reducer와 init을 훅이나 컴포넌트 외부에서 정의할 수 있음
  - `useState`에서는 초기 상태를 계산하는 용도로만 사용 가능 `useReducer`에서 처럼 인자로 함수를 전달할 수 없음

```tsx
const init = count => ({ count })
const reducer = (prev, delta) => ({ ...prev, count: prev.count + delta })

const ComponentWithUseReducer = ({ initialCount }) => {
  const [state, dispatch] = useReducer(reducer, initialCount, init)

  return (
    <div>
      {state.count}
      <button onClick={() => dispatch(1)}>+1</button>
    </div>
  )
}

const ComponentWithUseState = ({ initialCount }) => {
  const [state, setState] = useState(() => init(initialCount))
  const dispatch = delta => setState(prev => reducer(prev, delta))

  return (
    <div>
      {state.count}
      <button onClick={() => dispatch(1)}>+1</button>
    </div>
  )
}
```

- 인라인 리듀서 사용하기
  - `useReducer`에서는 외부 변수에 의존하는 형태로 인라인 리듀서 함수를 사용할 수 있음
  - 일반적으로 사용되지 않으며 꼭 필요한 경우가 아니라면 권장하지 않음

```tsx
const useScore = bonus => useReducer((prev, delta) => prev + delta + bonus, 0)
```

## (+) Concurrent React

> 동시성이라는 개념이 리액트에서 중요한 것 같아 이번 기회에 v18 이전부터 논의되어 v18에 도입된 동시성 개념에 대해 좀 더 알아봤습니다. (이미 v19 얘기가 나오고 있는 시점이지만...😅)

UI의 부드러운 상호작용과 효율적인 리소스 관리를 목표로 리액트 v18에서 본격적으로 제안된 렌더링 방식

```text
Async rendering → Concurrent Mode → Concurrent features(v18)
```

#### 동시성 렌더링의 이점

- [스크린 티어링](https://en.wikipedia.org/wiki/Screen_tearing) 방지: 화면이 갱신될 때 중간 상태가 보이는 문제를 줄여 부드러운 화면 전환 구현
- 우선순위 보호: 사용자 입력과 같은 중요한 작업을 비동기 작업보다 우선 처리
- 렌더링 최적화: I/O 대기 시간이나 유휴 시간 동안 사전 렌더링을 통해 앱 성능 향상
- 애니메이션 및 비디오 재생 개선: 부드러운 프레임 전환 및 성능 개선

#### 동시성 렌더링을 적용한 리액트 렌더링

- Render Phase: Virtual DOM 기반으로 컴포넌트 트리를 재구성하는 단계

  - React v18 이후 Concurrent Rendering 도입으로, 이 단계는 청크 단위로 작업을 처리하며, 우선순위가 높은 작업이 있다면 메인 스레드를 양보(Cooperative Scheduling)할 수 있음
  - 메인 스레드가 다른 작업을 처리할 수 있도록 작업을 중단(yield)할 수 있기 때문에, 이 단계에서는 API 호출, DOM 변경 등 부작용(Side Effects)을 유발하는 작업은 지양 → 이러한 작업은 반드시 Commit Phase에서 처리해야 함

> Virtual DOM 생성 → 비교(Diffing) → 재조정(Reconciliation)

- Commit Phase: Render Phase에서 계산된 변경 사항을 실제 DOM에 적용하는 단계
  - 이 단계에서는 DOM 업데이트가 동기적으로 실행되며, `useEffect`나 `useLayoutEffect` 같은 effect 관련 로직도 실행됨
  - Commit Phase는 항상 중단 없이 끝까지 실행됨

#### 전역 상태의 불일치 문제

> 동시성 렌더링에서 전역 상태의 불일치로 인해 [tearing](https://github.com/reactwg/react-18/discussions/69)이 발생할 수 있음
>
> - tearing: _"찢어짐"_, UI가 동일한 상태에 대해 서로 다른 값을 표시하는 경우
>   ![image.png](https://user-images.githubusercontent.com/2440089/124805949-29edc180-df2a-11eb-9621-4cd9c5d0bc5c.png)

- Mutable Store 문제
  - 전역 상태가 빠르게 변경될 경우, 상태를 읽는 컴포넌트에서 데이터 불일치 발생 가능
  - 해결 방안: (1) 동기적 렌더링 활용 / (2) 상태를 불변성을 지켜 관리

#### 주요 Concurrent Features

> [React v18.0](https://ko.react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)를 바탕으로 간단히 정리

- Strict Mode: Concurrent Features의 작동을 테스트하고 디버깅하는 모드로 `Strict Mode`에서 발생하는 두 번의 렌더링은 개발 모드에서만 동작
- 자동 Batching: 여러 상태 업데이트를 하나의 렌더링 사이클로 묶어서 처리

```tsx
setCount(count + 1)
setFlag(true)
// 기존: 두 번 렌더링 → v18: 한 번의 렌더링으로 처리
```

- Transitions: 긴급 업데이트(버튼 클릭, 입력)와 긴급하지 않은 업데이트(UI 애니메이션)를 구분하기 하여 UI 성능 개선
- 우선순위 관련 훅과 API → UI 우선순위 관리 및 비동기적 작업 처리 최적화
  - `useTransition`, `startTransition`: 낮은 우선순위 작업을 처리하는 훅과 함수
  - `useDeferredValue`: 비동기적으로 값을 지연 처리하여, 긴 렌더링 작업 동안에 UI를 빠르게 업데이트할 수 있도록 지원
