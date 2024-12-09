### 모듈 상태의 한계

모듈 상태는 리액트 컨포넌트 외부에 존재하는 전역으로 정의된 싱글턴이기 때문에 컴포넌트 트리나 하위 트리마다 다른 상태를 가질 수 없다.

컨텍스트와 구독을 함께 사용하여 전역 상태를 구현하면 컨테스트로 인해 하위 트리에서 상태를 분리할 수 있고, 구독으로 인해 리렌더링 문제를 피할 수 있다.

### 구독과 Context API를 함께 사용한 전역상태 구현부

StoreProvider에서 useRef를 사용하여 첫번째 렌더링시에 스토어 객체가 한 번만 초기화 되도록 한다.

```tsx
import { useRef } from "react";

const StoreProvider = ({
  initialState,
  children,
}: {
  initialState: State;
  children: React.ReactNode;
}) => {
  const storeRef = useRef<Store<State>>(null);

  if (!storeRef.current) {
    storeRef.current = createStore<State>(initialState);
  }

  return <Store.Provider value={store}>{children}</Store.Provider>;
};
```

useSelector 에서는 useSubScription을 사용하여 리렌더링을 방지한다. 이를 통해 컨텍스트와 구독의 이점을 동시에 누릴 수 있습니다.

```tsx
const useSelector = <S extends unknown>(selector: (state: State) => S) => {
  const store = useContext(Store);
  return useSubScription(
    useMemo(() => ({
      getCurrentValue: () => selector(store.getState()),
      subscribe: store.subscribe,
    })),
    selector
  );
};
```

### 리액트 전역 상태 라이브러리가 되려면

1. 리렌더링 최적화 문제

- 특정 컴포넌트가 전역 상태의 일부만을 필요로 할 때 전역상태의 다른 값이 바뀌었을 때 해당 컴포넌트가 리렌더링 되어서는 안된다.

2. 전역 상태에 값을 갱신할 때의 문제

- 전역 상태가 변경은 리액트 컴포넌트는 리렌더링을 일으켜야 한다.
- 이러한 상태 변경은 함수로서 제공되어야 하며 또한 변수가 직접 변경될 수 없도록 보장하여야 한다. ex) 클로저 활용

### Redux vs XState

리액트 뿐만 아니라 범용적인 상태관리를 제공하는 라이브러리로 Redux는 단방향 데이트 흐름을 통한 접근방식을 XState는 상태 머신 기반 접근 방식을 제공하고 있다.

Redux는 비교적 익숙한 반면 XState는 잘 사용해보지 않았다.

요새는 React-Query로 서버상태를 이관하는 추세여서 Redux 보다는 좀 더 가벼운 전역상태 관리 라이브러리를 선호하는 편인거 같은데 저는 여전히 Redux도 잘만 쓰면 이점이 있다고 생각해서 Redux를 사용하고 있는데요. Redux가 여전히 쓸만한다고 다른분들도 생각하는지? 아니면 Redux는 대체적으로 혐오하는 편인지 궁금하네요.

### 상태관리, 데이터 중심 vs 컴포넌트 중심

데이터 중심 접근 방식의 경우 모듈 상태가 리액트 외부의 자바스크립트 메모리에 있기 때문에 모듈 상태를 사용하는 편이 적합하며 이는 리액트 컴포넌트가 언마운트 된 뒤에도 존재할 수 있습니다.

반면 컴포넌트 중심의 접근 방식은 컴포넌트 생명 주기 내에서 (마운트 > 리렌더링 > 언마운트) 전역 상태를 함께 유지하는 것이 더 적합하며 컴포넌트가 언마운트 되면 전역 상태도 함께 사라져야 하고, 이때 자바스크립트 메모리에 두 개 이상의 동일한 전역 상태를 둘 수 있습니다.

물론 위 두 접근 방식은 꼭 둘 중 하나만 선택해야 하는 것은 아닙니다.

예를 들면 jotai의 [Provider API](https://jotai.org/docs/core/provider)를 보면 하위 트리에서 얼마든지 Provider를 만들어서 위와같은 기능을 제공합니다.

Providers are useful for three reasons:

1. To provide a different state for each sub tree.
2. To accept initial values of atoms.
3. To clear all atoms by remounting.

```tsx
const SubTree = () => (
  <Provider>
    <Child />
  </Provider>
);
```

마침 jotai를 사용하는 중인 회사코드에서 리팩토링할만한 부분를 찾아서 적용해 보았습니다.

아래 Chat 컴포넌트의 생명주기와 전역상태인 `modeAtom`를 동일하게 만들어주기 위해 클린업 함수를 사용하고 있는데

```tsx
interface Props {
  channel: Channel;
}

const Chat: FC<Props> = ({ channel }) => {
  const [mode, setMode] = useAtom(modeAtom);
  useEffect(
    // 언마운트시에 mode atom 초기화가 필요합니다.
    () => () => {
      if (mode !== "none") {
        setChatMode("none");
      }
    },
    [chatMode, setChatMode]
  );

  return <Main>{...생략}</Main>;
};

export default Chat;
```

아래와 같이 Provider를 감싸주는 것만으로 컴포넌트 생명주기와 동일하게 atom 초기화가 가능합니다.

```tsx
interface Props {
  channel: Channel;
}

const ChatContent: FC<Props> = ({ channel }) => {
  const chatMode = useAtomValue(chatModeAtom);
  return <Main>{...생략}</Main>;
};

const Chat: FC<Props> = ({ channel }) => {
  return (
    <Provider>
      <Main>
        <ChatContent channel={channel} />
      </Main>
    </Provider>
  );
};

export default Chat;
```

하지만 SubTree내에서와 그 상위트리에서의 상태가 불일치가 일어나고 필요한건 전역 atom의 초기화 였기때문에 해당 코드는 폐기되었습니다…

### 리렌더링 최적화 문제

책에서는 3가지 방법을 제안하고 있습니다.

1. 선택자 함수 사용
2. 속성 접근 감지
3. 아톰 사용

일반적으로 선택자 함수(useSelector)를 사용하지만 상태 사용 추적(state usate tracking) 방식도 소개하고 있습니다. 선택자 함수가 수동으로 최적화를 하는 반면 상태 사용 추적은 자동 최적화를 해준다고 합니다.

useTrackedState를 구현하려면 상태 객체에 대한 속성 접근을 확인하기 위한 프락시(proxy)가 필요합니다.

단 useSelector는 파생값을 만들어 상태를 간단한 값으로 나타낼 수 있습니다. 이런 경우 useSelector의 최적화가 좀 더 나은 경우 입니다.

```tsx
// isSmall이 바뀔때만 렌더링
const isSmall = useSelecter((state) => state.a < 10))

// state.a가 바뀔때마다 렌더링
const isSmall = useTrackedState().a < 10
```

아톰 사용

아톰(atom)은 리렌더링을 일으키는 최소 상태 단위로 전역상태를 store에서 동시에 관리하는 것이 아니라 아톰이라는 단위로 나누어서 개별적으로 관리하는 것을 말합니다.

### 질문거리

다음 장부터 상태관리 라이브러리 소개가 시작되는데 저희 회사에서는 redux + redux observable + jotai 조합으로 사용중입니다. 다들 상태관리에 어떤 라이브러리를 사용하고 계신가요?
