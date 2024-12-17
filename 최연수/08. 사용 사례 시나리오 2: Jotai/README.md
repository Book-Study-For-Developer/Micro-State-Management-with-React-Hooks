# 08. 사용 사례 시나리오 2: Jotai

> Jotai로 렌더링 최적화 하기, atom 값을 저장하는 방식 등 다양한 기능

<br/>

## 🔖 1. Jotai란?

### 특장점

- Jotai는 '원자(atom)'라는 가장 작은 상태 단위로 전역 상태를 정의하고, 필요한 컴포넌트에서 해당 atom만 사용하므로 상태 관리가 마치 `useState`를 전역으로 확장한 것처럼 자연스럽다.
- 따로 Provider를 쓰지 않고도 atom을 직접 컴포넌트에서 가져다 쓸 수 있다. 따라서 복잡한 Context 설정 없이도 전역 상태를 관리할 수 있다.
- 특정 atom만 읽는 컴포넌트는 해당 atom에만 반응한다. 다른 atom이 변해도 렌더링되지 않으므로 불필요한 렌더링이 줄어든다.
- atom을 비동기적 데이터 소스(예: fetch, async 함수)와 연결할 수 있어, 로딩 상태나 비동기 로직 처리에도 적합하다.
- 단점으로는 상태를 어디서 어떻게 atom으로 나눌지 설계가 필요하다는 점이 있겠다.

### 예시 코드

```jsx
// Jotai에서 제공하는 기본 함수: atom, useAtom
import { atom, useAtom } from 'jotai';

// 1. atom 정의: 전역 상태인 count를 관리할 atom
const countAtom = atom(0);

// 2. 이 atom을 사용하는 컴포넌트
function Counter() {
  // useAtom을 사용하면 [값, 업데이트함수]를 반환해,
  // 마치 useState처럼 count를 읽고 바꿀 수 있다.
  // useState(0) 대신, useAtom(countAtom)을 사용한다.
  const [count, setCount] = useAtom(countAtom);

  return (
    <div>
      <h1>유령이 얼마나 남았나요?: {count}</h1>
      <button onClick={() => setCount((c) => c + 1)}>유령 +1 증가</button>
      <button onClick={() => setCount((c) => c - 1)}>유령 -1 감소</button>
    </div>
  );
}

export default function App() {
  return (
    <div>
      {/* Counter를 렌더링하는 어디서든 countAtom을 통해 상태를 공유함 */}
      <Counter />
      <Counter />
    </div>
  );
}
```

<br/>

## 🔖 2. 하향식 접근 방법과 상향식 접근 방법

### 하향식(Top-down) 접근 방법

- 상위 레벨에서 전체 애플리케이션에 필요한 전역 상태 구조를 먼저 설계한 뒤, 그 구조에서 필요한 부분을 잘게 쪼개며 atom으로 정의하는 방식
- 초반에 전체적인 상태 모델(e.g. 전역 Store 등)을 어느정도 확정하고 시작하기에 큰 그림을 먼저 그리고 세부 atom을 나중에 채워 넣는 느낌
- 아무래도 상태가 복잡하거나, 프로젝트에서 전체적인 상태 모델 구조가 명확할 때 유용할 것 같다.
- 다만, 초반에 상태 모델 설계에 많은 고민이 필요하거나 나중에 구조가 바뀌면 일을 두 번해야 할 수도..

### 상향식(Bottom-up) 접근 방법

- 필요한 atom을 하나씩 만들고, 이 atom들을 점진적으로 조합하며 최종적으로 전역 상태 구조를 형성하는 방식
- 가장 작은 단위인 atom을 필요할 때마다 정의하고, 점차 이 atom들을 연결하거나 파생 atom을 만들면서 복잡한 상태를 형성한다.
- 요구사항 변화에 대응하기 쉬우나, 나중에 상태가 매우 많아지면 구조가 파편화될 수 있다.

### Jotai는 Bottom-up, zustand는 Top-down 방식?

- [몇몇 블로그 글](https://team-beat.tistory.com/21)들을 보니 Jotai는 Bottom-up, zustand는 Top-down이라고 한다. 개인적으로 Zustand에는 어느정도 해당되는 말이나, Jotai에 있어서는 꼭 맞는 말은 아닌 것 같다.
- Jotai의 경우, (책에서도 설명하듯) Jotai는 특정 접근 방식을 강제하지 않는다. 즉, Jotai는 Bottom-up과 Top-down 모두 구현할 수 있는 라이브러리다. 위에서 정리했듯 설계를 할 때 상태 구조를 어떻게 잡아가느냐에 따라 다르게 구현할 수 있는 것이다.
- 반면 Zustand는 store에 상태와 업데이트 로직을 모아두고, 컴포넌트에서 필요한 부분만 가져다 쓰는 방식이므로 Top-down에 가깝다고 볼 수 있을듯 하다.
- [공식 문서](https://jotai.org/docs/basics/comparison#:~:text=Traditionally%2C%20a%20top,need%20for%20memoization.)에서는 Jotai가 기존 Top-down 접근(선택자를 사용하고 memoization하는 방식)과 달리, Recoil의 영감을 받아 Bottom-up 접근이 자연스럽고 장점이 있다는 점을 말하고 있다.

<br/>

## 🔖 3. 배열 구조 추가하기

### Atoms-in-Atom

- Jotai에서 배열 형태의 상태를 관리할 때 배열 자체를 하나의 atom에 담고, 그 배열의 각 요소가 또 다른 atom들을 참조하는 형태
- 쉽게 말해, '배열(atom) 안에 원소 하나하나도 각각 atom으로 관리하는' 패턴이다.
- 이렇게 하면 배열의 특정 원소만 변경되었을 때, **그 원소에 해당하는 atom만 업데이트**되어 불필요한 리렌더링을 방지할 수 있다.
- 또한 배열에서 원소를 추가하거나 제거할 때도 해당 원소에 해당하는 atom만 조작하면 되므로 관리가 편해진다.

### 왜 이렇게 하는 걸까?

- 일반적으로 하나의 atom에 전체 배열을 담고 그 배열을 직접 변경하면, 배열 전체가 갱신되는 것으로 인식되어 배열 전체를 사용하는 컴포넌트가 다시 렌더링될 수 있다.
- 하지만 Atoms-in-Atom 방식에서는 각 원소를 별도의 atom으로 분리해, 그 원소를 필요로 하는 부분만 업데이트한다. 즉, 배열의 한 원소가 변해도 다른 원소를 참조하는 컴포넌트는 영향을 받지 않는다.

### 예시 코드

```jsx
import { atom, useAtom } from 'jotai';

// 1. 각 item을 위한 atom을 만든다고 생각해보자.
const createItemAtom = (initialValue) => atom(initialValue);

// 2. 전체 item 리스트 atom 안에 'item atom'들을 담는다.
const itemsAtom = atom([
  createItemAtom('Item 1'),
  createItemAtom('Item 2'),
  createItemAtom('Item 3'),
]);

function Item({ itemAtom }) {
  // 각각의 itemAtom을 직접 useAtom으로 사용 가능
  const [item, setItem] = useAtom(itemAtom);
  return (
    <div>
      <input
        value={item}
        onChange={(e) => setItem(e.target.value)}
      />
    </div>
  );
}

function ItemList() {
  const [itemAtoms, setItemAtoms] = useAtom(itemsAtom);

  const addItem = () => {
    // 새 item atom 추가
    setItemAtoms([...itemAtoms, createItemAtom(`Item ${itemAtoms.length + 1}`)]);
  };

  return (
    <div>
      <button onClick={addItem}>item 추가</button>
      <div>
        {itemAtoms.map((itemAtom, index) => (
          <Item key={index} itemAtom={itemAtom} />
        ))}
      </div>
    </div>
  );
}

// Item 컴포넌트는 itemAtom을 직접 받아 해당 아이템의 값만 업데이트한다.
// 다른 아이템과는 무관하므로, 이 아이템 값이 바뀌어도 다른 아이템들은 재렌더링할 필요가 없다.

// 쉽게 말해, Item 1의 atom 값만 변경되면, 그 atom을 구독하는 컴포넌트만 업데이트된다.
// 나머지 Item 2나 Item 3와 관련된 컴포넌트는 그대로 유지되어 리렌더링되지 않는다.

export default function App() {
  return (
    <div>
      <h1>Atoms-in-Atom 예시 코드</h1>
      <ItemList />
    </div>
  );
}
```
