# 09. 사용 사례 시나리오 3: valtio

> bundle size : 2.7 KiB | downloads : 30M

- 변경 가능한 갱신 모델을 기반으로 하는 전역 상태 관리 라이브러리
- 간단한 자기소개 : `makes proxy-state simple` / 프록시 기반 상태를 심플하게 만들어 준다
- 모듈 상태용(proxy)
- `Proxy`를 사용한 변경 불가능 스냅샷 생성
- `Proxy` 를 활용한 자동 렌더링 최적화 - 선택자가 필요없는 **상태 사용 추적**
  - 상태 부분 사용 감지 → 해당 부분만 리렌더링

## 사용하기

- `Proxy` 는 두번째 인자로 핸들러들을 객체에 담아 추가할 수 있고 특정 포인트에 트리거가 동작한다
- 이를 통해 변경 불가능한 객체를 만들 수 있고, React에서는 동작하지 않는 상태 업데이트 코드(직접적인 원본에 대한 변경)을 동작하게 할 수 있다.

```tsx
//스토어 만들기
const state = proxy({ count: 0 });

const snap1 = snapshot(state);
console.log(snap1); // { count: 0 }

++state.count;

const snap2 = snapshot(state);
console.log(snap2); // { count: 1 }

console.log(snap1 === state); // false
console.log(snap2 === state); // false
console.log(snap1 === snap2); // false
```

- state와 각 스냅샷은 서로 다른 참조를 가진다

## 최적화

- `useSnapshot` 훅을 통해 초반에 언급했던 상태 사용 추적이 가능해진다.