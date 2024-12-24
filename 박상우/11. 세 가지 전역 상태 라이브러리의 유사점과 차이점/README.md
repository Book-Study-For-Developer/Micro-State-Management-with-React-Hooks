## Zustand와 Redux의 차이점

zustand와 Redux는

- 단방향 데이터 흐름을 기반으로 한다.
- 상태를 갱신하는 명령을 나타내는 action이라는 개념을 활용하고, action 이후에는 새로운 상태 필요한 곳으로 전파된다.

  → dispatch와 전파를 구분한 것은 데이터 흐름을 단순화 하고 시스템을 예측 가능하게 한다.

- 상태 갱신 방식에 차이가 있다.
  - Redux는 리듀서를 기반으로 이전 상태와 action을 전달받아 Action을 수행하고 그 결과로 새로운 상태를 반환하는 순수함수를 활용한다.
    → 엄격한 방식이지만 예측 가능성이 높다.
  - Zustand에서는 리듀서를 사용할 필요가 없다.

<br />

### Redux와 Zustand를 사용한 예제

##### Redux

```tsx
// store
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from '../features/counter/counterSlice';

export const store = configureStore({
  reducer: {
    counter: counterReducer,
  },
});
```

```tsx
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

const initialState = {
  value: 0,
};

export const counterSlice = createSlice({
  // 액션 타입을 자동으로 생성할 때 name을 접두사로 활용. 이를 통해 동일한 이름의 액션이 서로 다른 slice에서 충돌하지 않도록 방지.
  name: 'counter',
  initialState,
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;

import { useSelector, useDispatch } from 'react-redux';
import { decrement, increment } from './counterSlice';

export function Counter() {
  // selector 함수를 통해 스토어에서 상태를 가져옴
  const count = useSelector(
    (state: { counter: { value: number } }) => state.counter.value
  );
  const dispatch = useDispatch();
  return (
    <div>
      <button onClick={() => dispatch(increment())}>Increment</button>
      <span>{count}</span>
      <button onClick={() => dispatch(decrement())}>Decrement</button>
    </div>
  );
}
```

```tsx
import { Provider } from 'react-redux';
import { store } from './app/store';
import { Counter } from './features/counter/Counter';

const App = () => (
  <Provider store={store}>
    <div>
      <Counter />
      <Counter />
    </div>
  </Provider>
);

export default App;
```

<br />

#### Zustand

```tsx
import create from 'zustand';

type State = {
	counter: {
		value: number;
	};
	counterAction: {
		increment: () => void;
		decrement: () => void;
		incrementByAmount: (amount: number) => void;
	}
}

export const useStore = create<State>((set) => ({
	// 상태
	counter: { value: 0 },
	// 액션
	counterAction: {
		increment: () =>
			set((state) => ({
				counter: { value: state.counter.value + 1 },
			})),
		decrement: () =>
			set((state) => ({
				counter: { value: state.counter.value - 1 },
			})),
		incrementByAmount: () =>
			set((state) => ({
				counter: { value: state.counter.value + amount },
			})),
	}
});

import { useStore } from './store';

export function Counter() {
	const count = useStore((state) => state.counter.value);
	const { increment, decrement } = useStore(
		(state) => state.counterActions
	)

	return (
		<div>
			<div>
				<button onClick={increment}>Increment</button>
				<span>{ count }</span>
				<button onClick={decrement}>Decrement</button>
			</div>
		</div>
	)
}

import { Counter { From './Counter';

const App = () => (
	<div>
		<Counter />
		<Counter />
	</div>
)

export default App;
```

<br />

#### 주요한 차이점

- 디렉토리 구조

  Redux의 경우 feature에 따라 구분하는 디렉토리 구조를 제안하고, createSlice 내부에서 디렉토리 구조와 동일하게 따르도록 설계하고 있다.

  ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8af0ac89-313c-4704-8adf-13f01ae8e670/d7c7579f-e6d6-41e9-b2b7-397418c08332/image.png)

  Zustand의 경우 별도의 패턴이 정해져 있지 않고, 이번 예시 또한 store 파일 만 활용하고 있다. Redux에서 사용헀던 feature 패턴을 활용할 수 있다.

  ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8af0ac89-313c-4704-8adf-13f01ae8e670/aad51d78-accd-47bc-b060-343bf337bb41/image.png)

- 스토어 생성 코드에서 Immer 사용 여부
  Immer를 활용하면 state.value += 1과 같이 속성에 접근하여 직접 갱신이 가능하다. Redux의 경우 기본적으로 Immer를 활용하고, Zustand에서는 활용하지 않는다. 하지만 Zustand에서도 Immer 활용이 가능하다.

  ```tsx
  import create from 'zustand';
  import { produce } from 'immer';

  type State = {
    counter: {
      value: number;
    };
    counterAction: {
      increment: () => void;
      decrement: () => void;
      incrementByAmount: (amount: number) => void;
    };
  };

  export const useStore = create<State>((set) => ({
    // 상태
    counter: { value: 0 },
    // 액션
    counterAction: {
      increment: () =>
        set(
          produce((state: State) => {
            state.counter.value += 1;
          })
        ),
      decrement: () =>
        set(
          produce((state: State) => {
            state.counter.value -= 1;
          })
        ),
      incrementByAmount: (amount: number) =>
        set(
          produce((state: State) => {
            state.counter.value += amount;
          })
        ),
    },
  }));
  ```

- 상태 전파

  Redux의 경우 컨텍스트를 활용하며, Zustand는 모듈 임포트를 활용한다. Zustand에서 컨텍스트를 선택적으로 사용할 수 있다.

  ```tsx
  import { createContext, useContext } from 'react';
  import create, { StoreApi } from 'zustand';

  // Zustand 스토어 타입
  type State = {
    counter: number;
    increment: () => void;
  };

  // Zustand 스토어 생성
  const createStore = () =>
    create<State>((set) => ({
      counter: 0,
      increment: () => set((state) => ({ counter: state.counter + 1 })),
    }));

  // Context 생성
  const ZustandContext = createContext<StoreApi<State> | null>(null);

  // Context Provider 컴포넌트
  export const ZustandProvider = ({
    children,
  }: {
    children: React.ReactNode;
  }) => {
    const store = createStore();
    return (
      <ZustandContext.Provider value={store}>
        {children}
      </ZustandContext.Provider>
    );
  };

  // 컨텍스트에서 store를 활용하는 훅
  export const useZustandStore = () => {
    const store = useContext(ZustandContext);
    if (!store)
      throw new Error('useZustandStore must be used within ZustandProvider');
    return store;
  };

  import { useZustandStore } from './ZustandContext';

  export const Counter = () => {
    const { counter, increment } = useZustandStore()((state) => ({
      counter: state.counter,
      increment: state.increment,
    }));

    return (
      <div>
        <p>Counter: {counter}</p>
        <button onClick={increment}>Increment</button>
      </div>
    );
  };

  import { ZustandProvider } from './ZustandContext';
  import { Counter } from './Counter';

  export default function App() {
    return (
      <ZustandProvider>
        <div>
          <h1>Zustand with Context</h1>
          <Counter />
        </div>
      </ZustandProvider>
    );
  }
  ```

  Zustand와 Context를 함께 활용하면 여러 컨텍스트를 활용하여 유연하게 전역 상태 관리가 가능하고, 특정 범위의 컴포넌트 트리에만 특정 전역 상태로 제한하여 사용할 수 있다.

- 상태 관리 구조
  Redux는 단방향 데이터 흐름을 기반으로 하기 때문에 상태 갱신을 위해서 항상 액션을 디스패치 해야한다. 반면 Zustand는 별도의 규칙이 존재하지 않기 때문에 Redux와 같은 단방향 흐름을 만들기 위해서는 별도의 처리를 개발자가 직접 해야한다.
  → Redux의 단점을 보일러 플레이트가 많다는 점을 꼽기도 하는데 이런 엄격한 단방향 데이터 흐름을 만들기 위해서 어쩔 수 없이 발생한 사이드 이펙트 처럼 느껴짐.
  → Zustand와 같이 가벼운 전역 상태 관리 도구들이 인기가 많아진 이유가 tanstack-query의 등장 배경 처럼 서버 상태와 클라이언트 상태를 구분하여 관리하고, 클라이언트에서 전역으로 관리할 상태가 이전보다 줄어들어 기존의 Redux를 그대로 활용하는 것이 구현 시간 대비 효과가 적어졌기 때문이 아닐까.🧐

<br />

## Jotai와 Recoil의 사용 시점

Jotai는 Recoil에서 많은 영감을 받아 만들어졌다.

### Recoil → Jotai 변환 예제

#### Recoil 예제

```tsx
import {
  RecoilRoot,
  atom,
  selector,
  useRecoilState,
  useRecoilValue,
} from 'recoil';

// 상태 정의, key가 필요함
const textState = atom({
  key: 'textState',
  default: '',
});

const TextInput = () => {
  // 상태 사용
  const [text, setText] = useRecoilState(textState);
  return (
    <div>
      <input
        type='text'
        value={text}
        onChange={(event) => {
          setText(event.target.value);
        }}
      />
      <br />
      Echo: {text}
    </div>
  );
};

// 파생 상태 정의
const charCountState = selector({
  key: 'charCountState',
  // 파생 상태 값 반환
  get: ({ get }) => get(textState).length,
});

const CharacterCount = () => {
  const count = useRecoilValue(charCountState);
  return <>Character Count: {count}</>;
};

const CharacterCounter = () => (
  <div>
    <TextInput />
    <CharacterCount />
  </div>
);

const App = () => (
  <RecoilRoot>
    <CharacterCounter />
  </RecoilRoot>
);

export default App;
```

<br />

#### Jotail 예제

```tsx
import { atom, useAtom } from 'jotai';

// 상태 정의, key가 필요없음
const textAtom = atom('');

const TextInput = () => {
  // 정의한 아톰 활용
  const [text, setText] = useAtom(textAtom);
  return (
    <div>
      <input
        type='text'
        value={text}
        onChange={(event) => {
          setText(event.target.value);
        }}
      />
      <br />
      Echo: {text}
    </div>
  );
};

// 파생 아톰
const charCountAtom = atom((get) => get(textAtom).length);

const CharacterCount = () => {
  const [count] = useAtom(charCountAtom);
  return <>Character Count: {count}</>;
};

const CharacterCounter = () => (
  <div>
    <TextInput />
    <CharacterCount />
  </div>
);

const App = () => (
  <>
    <CharacterCounter />
  </>
);

export default App;
```

<br/>

#### Recoil과 Jotai 비교

- key의 존재
  Jotai는 Recoil에서 아톰 정의를 위해 항상 사용해야하는 key를 활용하지 않도록 하여 보다 간편하게 아톰을 정의할 수 있게 되었다. 개발자 경험에 있어 중복을 고려하여 key 네이밍을 매번 하지 않아도 되기 때문에 편리하다. Jotail가 key 제거를 위해 weakMap을 활용했는데, 이로 인해서 Recoil에서 key를 텍스트로 관리함으로서 쉽게 가능했던 직렬화 부분에 추가적인 기법이 필요해졌다.
- atom 함수
  Jotail의 atom 함수는 Recoil에서의 atom과 selector 모두를 대체 가능하다. 하지만 두 함수의 모든 기능을 커버하기는 어렵기 때문에 추가적인 함수가 더 필요하다.
- Jotai 공급자 제거 모드
  Jotai는 Provider를 사용하지 않는 기능이 있다. 라이브러리 사용성에 있어 보다 개발자 친화적인 기능이다.

<br />

## Valtio와 Mobx의 사용

Valtio와 Mobx는 모두 변경 가능한 상태를 기반으로 하여 개발자가 직접 상태를 변경할 수 있다. 그리고 객체 변경 로직이 자연스럽고 간결하다.

Valtio는 훅을 통해 랜더링 최적화를 하는 반면, Mobx는 고차 컴포넌트(HOC : Higher-order Component)를 활용한다.

> **Valtio와 Immer**
> Valtio와 Immer는 불변 상태와 변경 가능한 상태를 활용한다는 점에서 공통점 있다. Valtio는 변경 가능한 상태를 불변 상태로 변환하여 활용하지만, Immer는 불변 상태를 변경 가능한 상태로 일시적으로 활용하는 방식을 사용하고 있다.

<br />

### Mobx, Valtio 예제

#### Mobx 예제

```tsx
import { makeAutoObservable } from 'mobx';
import { observer } from 'mobx-react';

class Timer {
  secondsPassed = 0;

  constructor() {
    // 생성한 Timer 인스턴스 자체를 관찰 가능한 객체로 활용
    makeAutoObservable(this);
  }

  increase() {
    this.secondsPassed += 1;
  }

  reset() {
    this.secondsPassed = 0;
  }
}

const myTimer = new Timer();

setInterval(() => {
  myTimer.increase();
}, 1000);

// 고차 컴포넌트 활용.
// timer의 secondsPassed가 변경될 때 마다 리랜더링이 발생한다.
const TimerView = observer(({ timer }: { timer: Timer }) => (
  <button onClick={() => timer.reset()}>
    Seconds passed: {timer.secondsPassed}
  </button>
));

const App = () => (
  <>
    <TimerView timer={myTimer} />
  </>
);

export default App;
```

<br />

#### Valtio 예제

```tsx
import { proxy, useSnapshot } from 'valtio';

const myTimer = proxy({
  secondsPassed: 0,
  increase: () => {
    myTimer.secondsPassed += 1;
  },
  reset: () => {
    myTimer.secondsPassed = 0;
  },
});

setInterval(() => {
  myTimer.increase();
}, 1000);

const TimerView = ({ timer }: { timer: typeof myTimer }) => {
  // 상태 활용에 따라 사용된 부분이 변경될 때 리랜더링이 발생
  const snap = useSnapshot(timer);
  return (
    <button onClick={() => timer.reset()}>
      Seconds passed: {snap.secondsPassed}
    </button>
  );
};

const App = () => (
  <>
    <TimerView timer={myTimer} />
  </>
);

export default App;
```

<br />

### Mobx와 Valtio 예제 비교

- 갱신 방식
  두 방식 모두 변경 가능한 상태를 사용하지만 Mobx는 클래스 기반, Valtio는 객체 기반이다.
  Valtio는 정해진 스타일이 없으며 아래와 같이 상태와 함수를 분리하는 것도 가능하다.

  ```tsx
  const timer = proxy({ secondsPassed: 0 });

  export const increase = () => {
    timer.secondsPassed += 1;
  };

  export const reset = () => {
    timer.secondsPassed = 0;
  };

  export const useSecondsPasses = () => useSnapshot(timer).secondsPassed;
  ```

  이렇게 구현했을 때는 코드 분할, 최소화, 불필요한 코드 제거 등이 가능해 번들 사이즈 최적화도 가능하다는 점이 있다.

- 렌더링 최적화 방식
  Mobx는 옵저버 방식을 활용하고, Valtio는 훅 방식을 활용한다.
  옵저버 방식이 더 예측 가능성이 높고, 훅 방식은 동시성 랜더링에 효과적이다. 접근 방식에 따라 구현 방식이 상이 할 수 있다.

## Zustand, Jotai, Valtio 비교

세 라이브러리는 모두 같은 개발자 집단(Poimandres)에서 나온 상태 관리 도구이다. 세 라이브러리는 작은 API를 제공하고, 필요에 따라 다른 라이브러리와 조합해서 사용할 수 있도록, 서로 다른 방식으로 구현되어있다는 특징이 있다.

- 상태가 어디에 위치하는가?
  크게 모듈 상태와 컴포넌트 상태가 존재한다.
  - 모듈 상태 : 모듈 수준에서 생성되는 상태로, 리액트에 속하지 않는 상태
    ( Zustand, Valtio )
  - 컴포넌트 상태 : 리액트 컴포넌트 생명 주기에서 생성되고 리액트에 의해 제어되는 상태
    ( Jotai )
    Jotai에서 createAtom을 통해 생성되는 아톰 객체는 실제 값을 가지진 않는다. 실제 아톰 값은 Provider를 통해 컴포넌트에 저장되고, 이 덕분에 여러 컴포넌트에서 사용가능하다. 이런 동작을 모듈 상태로 활용하기는 어렵다. 모듈 상태를 사용하는 Zustand와 Jotai를 통해 구현하고 싶다면 결국 리액트 컨텍스트를 사용하게 된다.
    반면 리액트 외부에서 컴포넌트 상태에 접근하는 것은 기술적으로 불가능하기 때문에 컴포넌트 상태와 연결을 위해 모듈 상태를 사용해야할 가능성이 있다.
    애플리케이션의 요구사항에 따라 모듈 상태와 컴포너트 상태 중 어떤 것을 사용할지 결정해야 한다. 일반 적으로 전역 상태에 대해 모듈 상태 또는 컴포넌트 상태 중 하나를 사용하면 대체로 요구사항을 대체로 충족하지만, 드물게는 두 유형의 상태를 모두 사용하는 것이 합리적일 수 있다.
- 상태 갱신 스타일이 어떠한가?
  Zustand는 불변 상태 모델을 기반으로 하지만, Valtio는 변경 가능 모델을 기반으로 한다.
  불변 상태 모델에서는 생성된 상태를 변경할 수 없기 때문에 새로운 객체를 만드는 방식으로 업데이트가 이루어진다. 변경 가능 모델은 객체에 접근하여 본질적으로 변경이 가능하다. 불변 모델은 객체 참조를 비교해서 변경 사항이 있는지 파악하기 쉽다는 점이다. 중첩된 객체에 대한 성능을 높이는데 도움이 된다.
  > Valtio vs (Zustand, Jotai) + Immer
  > Zustand와 Immer를 활용하여 Valtio와 비슷하게 상태에 직접 접근하여 상태를 바꿀 수 있다. 하지만 Valtio가 더 최적화되어 있는 방식이며 API 크기가 더 작다. 큰 객체를 다룰 때 Jotai + Immer가 효과적이고, Jotai도 Immer와의 호환성을 위한 기능을 제공한다. 하지만 Jotai의 컨셉상 atom의 크기가 크지 않기 때문에 불변 상태 모델 활용하기 어려워 Immer를 추가하는 경우는 드물다.
