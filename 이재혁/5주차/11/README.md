# 11. 세 가지 전역 상태 라이브러리의 유사점과 차이점

## 1. Zustand vs Redux

### 유사

- Flux 패턴 - 단방향 데이터 흐름
  - Action → Dispatcher → Store → View → Action - Dispatcher → …
- 예측 유용한 흐름도

### 차이

- 상태를 갱신하는 방법
  - Redux
    - Reducer 기반 (prev 상태와 action 객체를 받아 새로운 상태를 반환하는 순수 함수)
    - 꽤나 번거롭지만 예측 가능성 높음
  - Zustand
    - Reducer가 반드시 필요한 것은 아님
    - 메소드로서의 기능으로도 상태를 업데이트하기 충분
    - 불변이기에 새로운 상태를 반환한다.
- Immer의 사용 유무
  - Redux
    - `상태의 직접적인 변경을 허용한다` 처럼 보이지만 아님
      > Immer (German for: always) is a tiny package that allows you to work with immutable state in a more convenient way.

## 2. Recoil vs Jotai

- 처음에는 의도적으로 Recoil에서 Jotai로 마이그레이션하는 데 도움이 되도록 설계
- Recoil is dead…
- `기본으로 key값의 필요 유무` 라는 차이점 이외에는 동일하다

```tsx
// Jotai
const textAtom = atom("");

// Recoil
const textState = atom({
  key: "textState",
  default: "",
});
```

- 파생된 값을 정의 하기 위한 방법

```tsx
// Jotai
const charCountAtom = atom((get) => get(textAtom).length);

// Recoil
const charCountState = atom({
  key: "charCountState",
  default: ({ get }) => get(textState).length,
});
```

## 3. MobX vs Valtio

### 유사

- 둘 다 변경 가능한 상태 기반
- Valtio vs Immer
  - 둘 다 `불변 상태 <-> 변경 가능한 상태` 연결 노력
  - Valtio - 기반 : 변경 가능한 상태 / 불변 상태로 변환
  - Immer - 기반 : 불변 상태 / 변경 가능한 상태로 변환

### 차이

- 렌더링 최적화의 차이
  - Valtio - 훅을 사용
  - MobX - HoC 사용
- 갱신 방식의 차이
  - 변경 가능한 상태를 사용
    - MobX : 클래스 기반
    - Valtio - 객체 기반
  - Valtio의 경우 그렇기에 상태 객체에서 함수를 분리할 수 있다.
    - proxy 함수로 만든 상태 객체 외부에서 갱신 함수를 정의하는 것이 가능하다. (p.236)
    - 이 방법의 장점은 코드 분할과 최소화, 불필요한 코드 제거가 가능하다는 것이다.
      > ❓ 그 말인즉슨, 모듈 형태의 함수는 빌드 과정에서 사용하지 않는 코드는 제거된다는 것?

## Zustand, Jotai, Valtio 비교하기

### 1. 상태가 어디에 위치하는가??

- 리액트에서 상태를 다루는 2가지 방식
  1. 모듈 상태 - 모듈 수준에서 생성되는 상태 (리액트에 속하지 X)
     1. e.g.) Zustand (create), Valtio (proxy)
  2. 컴포넌트 상태 - 리액트 컴포넌트 생명 주기에서 생성되고 리액트에 의해 제어되는 상태
     1. e.g.) Jotai (atom)

### 2. 상태 갱신 스타일은 무엇인가?

- Zustand - 불변 상태 모델 → 갱신을 위해서는 새로운 값을 리턴해야 한다. (리액트와 맞음)
- Valtio - 변경 가능한 상태 모델 → 갱신을 위해 해당 값을 직접 수정할 수 있다.
