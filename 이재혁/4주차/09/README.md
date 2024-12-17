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
- state와 각 스냅샷은 서로 다른 참조를 가진다

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

- 이렇게 proxy 내에서 액션도 같이 담을 수 있다.
- snap의 경우 snapshot으로 이루어진 `proxy`들은 `Object.freeze()` 메소드로 인해 변경 불가능해 진다
- snapshot을 통해 생성된 스냅샷은 프록시나 자식 프록시의 변경 사항이 발생할 떄 다시 생성된다.
-

```tsx
const state = proxy({
  count: 0,
  inc() {
    ++this.count;
  },
});

const Valtio = () => {
  const snap = useSnapshot(state);

  const handleClick = () => {
    state.inc(); // 가능
    snap.inc(); // 불가능 -> 변경 불가능한 상태이기 때문 // 딱히 에러를 뱉지는 않는다.
  };
  return (
    <div>
      {state.count}
      <button onClick={handleClick}>click me!</button>
    </div>
  );
};
```

- 프록시에 대한 모든 변경 사항이 아닌 특정하게 액세스한 키(또는 하위 구성 요소)가 변경된 경우에만 다시 렌더링된다.

```tsx
const createSnapshotDefault = <T extends object>(
  target: T,
  version: number
): T => {
  const cache = snapCache.get(target);
  if (cache?.[0] === version) {
    return cache[1] as T;
  }
  const snap: any = Array.isArray(target)
    ? []
    : Object.create(Object.getPrototypeOf(target));
  markToTrack(snap, true); // mark to track
  snapCache.set(target, [version, snap]);
  Reflect.ownKeys(target).forEach((key) => {
    if (Object.getOwnPropertyDescriptor(snap, key)) {
      // Only the known case is Array.length so far.
      return;
    }
    const value = Reflect.get(target, key);
    const { enumerable } = Reflect.getOwnPropertyDescriptor(
      target,
      key
    ) as PropertyDescriptor;
    const desc: PropertyDescriptor = {
      value,
      enumerable: enumerable as boolean,
      // This is intentional to avoid copying with proxy-compare.
      // It's still non-writable, so it avoids assigning a value.
      configurable: true,
    };
    if (refSet.has(value as object)) {
      markToTrack(value as object, false); // mark not to track
    } else if (proxyStateMap.has(value as object)) {
      const [target, ensureVersion] = proxyStateMap.get(
        value as object
      ) as ProxyState;
      desc.value = createSnapshotDefault(
        target,
        ensureVersion()
      ) as Snapshot<T>;
    }
    Object.defineProperty(snap, key, desc);
  });
  return Object.preventExtensions(snap);
};
```

- 파생 값 만들기
  - 기존 스냅샷이 프록시의 상태 변경 이후에 영향을 받지 않는 이유
  - 객체 getter의 값이 스냅샷 프로세스가 돌 때 복사가 된다고 한다.

```tsx
const state = proxy({
  count: 1,
  inc() {
    ++this.count;
  },
  get doubled() {
    return this.count * 2;
  },
});

console.log(state.doubled); // 2

// 1. 스냅샷을 떠서 상태 가져오기
const snap = snapshot(state);
console.log(snap.doubled); // 2

// 2. 프록시의 상태 변경
state.count = 10;

// 3. 하지만 바뀌지 않는 기존 스냅샷
console.log(snap.doubled); // 2
```

### 리액트에서 벗어나기

- valtio에서는 컴포넌트 외부에서도 상태에 접근할 수 있게 설계되어있다.
- 구독 기반으로 구성되어 있다.

```tsx
import { proxy, useSnapshot, subscribe } from "valtio";

const state = proxy({ count: 0, text: "hello" });
// 어디선가 많이 봐온...
const unsubscribe = subscribe(state, () =>
  console.log("state has changed to", state)
);
unsubscribe();
```

- 부분 상태도 구독이 가능하다

```tsx
const state = proxy({ obj: { foo: "bar" }, arr: ["hello"] });

// state.obj 만 구독하기
subscribe(state.obj, () => console.log("state.obj has changed to", state.obj));
state.obj.foo = "baz";

// state.arr 만 구독하기
subscribe(state.arr, () => console.log("state.arr has changed to", state.arr));
state.arr.push("world");
```

- watch라는 유틸 함수를 통해서도 구독과 동일한 역할을 하면서 여러 프록시를 구독할 수 있다.
  - 프록시가 변경되면 콜백이 실행된다.
  - 초기 watch가 한번 호출되어 초기화를 한번 해준다.

```tsx
export declare function watch(
  callback: WatchCallback,
  options?: WatchOptions
): Cleanup;
// use ================================================
import { proxy } from "valtio";
import { watch } from "valtio/utils";

const userState = proxy({ user: { name: "Juuso" } });
const sessionState = proxy({ expired: false });

watch((get) => {
  // `get` adds `sessionState` to this callback's watched proxies
  get(sessionState);
  const expired = sessionState.expired;
  // Or call it inline
  const name = get(userState).user.name;
  console.log(`${name}'s session is ${expired ? "expired" : "valid"}`);
});
// 'Juuso's session is valid'
sessionState.expired = true;
// 'Juuso's session is expired'
```

## 최적화

- `useSnapshot` 훅을 통해 초반에 언급했던 상태 사용 추적이 가능해진다.
- 대부분 Proxy 기반의 라이브러리들은 동등 비교 작업하는 데 있어 [`proxy-compare`](https://github.com/dai-shi/proxy-compare) 를 쓰는 듯 하다.
