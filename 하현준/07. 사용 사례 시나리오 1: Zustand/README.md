## Zustand 소개

https://zustand-demo.pmnd.rs/

현재는 `v5`으로 배포된 상태

<aside>
<img src="/icons/report_blue.svg" alt="/icons/report_blue.svg" width="40px" />

**`v5`에서 달라진 점? (https://github.com/pmndrs/zustand/releases)**

- No new features
- Drop many old things
- Migration from v4 should be smooth.

특별히 새로운 feature는 없고 오래된 것들을 제거하고, v4에서 마이그레이션을 쉽게 할 수 있도록 함

</aside>

## 사용법 ?

```tsx
import { create } from "zustand";

type State = {
  firstName: string;
  lastName: string;
};

type Action = {
  updateFirstName: (firstName: State["firstName"]) => void;
  updateLastName: (lastName: State["lastName"]) => void;
};

// Create your store, which includes both state and (optionally) actions
const usePersonStore = create<State & Action>((set) => ({
  firstName: "",
  lastName: "",
  updateFirstName: (firstName) => set(() => ({ firstName: firstName })),
  updateLastName: (lastName) => set(() => ({ lastName: lastName })),
}));

// In consuming app
function App() {
  // "select" the needed state and actions, in this case, the firstName value
  // and the action updateFirstName

  // 필요한 상태만 "select" 해서 리렌더링을 피해야 한다.
  const firstName = usePersonStore((state) => state.firstName);
  const updateFirstName = usePersonStore((state) => state.updateFirstName);

  return (
    <main>
      <label>
        First name
        <input
          // Update the "firstName" state
          onChange={(e) => updateFirstName(e.currentTarget.value)}
          value={firstName}
        />
      </label>

      <p>
        Hello, <strong>{firstName}!</strong>
      </p>
    </main>
  );
}
```

<aside>
<img src="/icons/report_blue.svg" alt="/icons/report_blue.svg" width="40px" />

set 은 내부적으로 Object.assign() 으로 구현되어 있다.

https://github.com/pmndrs/zustand/blob/main/src/vanilla.ts#L64-L79

```tsx
const setState: StoreApi<TState>["setState"] = (partial, replace) => {
  // TODO: Remove type assertion once https://github.com/microsoft/TypeScript/issues/37663 is resolved
  // https://github.com/microsoft/TypeScript/issues/37663#issuecomment-759728342

  // 바꿀 상태가 함수인지 아닌지에 따른 분기처리
  const nextState =
    typeof partial === "function"
      ? (partial as (state: TState) => TState)(state)
      : partial;

  // 만약 값이 바뀌었다면?
  if (!Object.is(nextState, state)) {
    // 이전 상태를 기억해둔다.
    const previousState = state;

    // 현재 상태를 새롭게 바꿀 값으로 지정
    state =
      replace ?? (typeof nextState !== "object" || nextState === null)
        ? (nextState as TState)
        : Object.assign({}, state, nextState);

    // 구독 중인 것들을 모두 실행
    listeners.forEach((listener) => listener(state, previousState));
  }
};
```

</aside>

### Zustand 사용할 때 최적화하기

```tsx
// const { firstName, lastName, updateFirstName } = usePersonStore
// 위의 코드로 가져오게 된다면, lastName이 변경될 때에 리렌러링이 발생하게 된다.
const firstName = usePersonStore((state) => state.firstName);
const updateFirstName = usePersonStore((state) => state.updateFirstName);
```

이렇게 선택자 기반 리렌더링 제어를 **수동 렌더링 최적화**라고 한다.

## 읽기 상태와 갱신 상태 사용하기

읽기 상태와 갱신 상태라는 용어를 사용하는데 말이 어렵게 느껴지는 것 같다.

Zustand에서 소개한 State, Action으로 보는게 더 편한 것 같다.

- 읽기 상태: 읽는데만 쓰이는 상태
- 갱신 상태: 상태를 갱신하는데 쓰이는 것

**파생 상태를 다루는 방법**

여기서 파생 상태란 상태로 부터 만들어지는(파생되는) 상태를 말한다.

```tsx
const selectCount1 = (state: StoreState) => state.count1;
const selectCount2 = (state: StoreState) => state.count2;

const Counter2 = () => {
  const count1 = useStore(selectCount1);
  const count2 = useStore(selectCount2);

  return <div>total: {count1 + count2}</div>;
};
```

위의 코드로 `total` 값을 구하게 되면 에지 케이스가 발생하게 된다.

책에서는 count1이 N만큼 증가하고, count2와 N만큼 감소할 때 리렌더링이 발생하는 Edge 케이스가 있다고 했다. ⇒ 사실 무슨 말인지 이해가 안가서 이대로 GPT에게 물어보았다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/253d1ac1-0c8d-4179-8d90-21ade38e0aea/93562fe4-284d-4f15-9a25-a92dc4ba3803/image.png)

😰

즉, 책에서 말하는 것은 **두 값이 변했지만 총합은 변화가 없었는데에도 불구하고 리렌더링이 발생하는 상황**을 말하는 것이다. 이런 엣지 케이스를 해결하기 위해서는 새로운 샐럭터를 사용할 것을 권장한다.

**방법1.**

```
const selectTotal = (state: StoreState) => state.count1 + state.count2;
```

또는 증감할 때 `total`까지 같이 관리하도록 하여 관리포인트를 줄이는 방법도 소개한다.

```tsx
const useStore = create<StoreState>((set) => ({
  count1: 0,
  count2: 0,
  total: 0,
  inc1: () =>
    set((prev) => ({
      ...prev,
      count1: prev.count1 + 1,
      total: prev.count1 + 1 + prev.count2,
    })),
  inc2: () =>
    set((prev) => ({
      ...prev,
      count2: prev.count2 + 1,
      total: prev.count2 + 1 + prev.count1,
    })),
}));
```

⇒ 오.. 😮 후자의 방법도 괜찮아 보인다. **핵심 포인트는 동기화를 여러 곳에서 다루지 않고 한번에 다루는 것!**

### 더 복잡한 데이터를 다뤄보기

Todo를 Zustand를 사용해 구현한 예시를 설명하는데 특별한 방법은 없이 흔히 아는 방식으로 하기 때문에 예제를 직접 살펴보는 게 나은 듯 해서 링크만 남긴다.

https://github.com/wikibook/msmrh/blob/main/chapter07/02_todo_example/src/App.tsx

## Zustand의 특징

- 라이브러리의 번들 크기가 매우 작다
- 선택자를 이용해 수동 렌더링 최적화를 해야 한다.
  - 수동 렌더링 최적화를 해야 하기 때문에 객체 참조 동등성을 이해해야 한다.
  - 선택자 코드를 작성해야 하므로 보일러플레이트 코드가 많아질 수 있다.

결론: 작은 번들 라이브러리 사용하고 싶다. 메모이제이션에 익숙하거나 수동 렌더링 최적화를 선호할 경우 Zustand 라이브러리를 추천

❓ : 회사에서도 Zustand를 사용중인데, 이런식으로 select해서 뽑아 쓰는게 반복된다고 느끼긴 했는데요.
이러한 반복 작업(보일러플레이트 코드)를 줄이는 좋은 방법이 있나요?
