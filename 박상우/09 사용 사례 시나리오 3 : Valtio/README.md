## Valtio 살펴보기

모듈 상태를 관리하는 Zustand는 불변 객체의 불변성을 통해 리액트 랜더링을 최적화한다.

객체 불변성을 사용하지 않고 `++moduelState.count` 와 같이 갱신이 가능하다면, setState함수와 같은 별도의 갱신 함수가 필요하지 않을 것이다.

선택 속성만 변경했을 때 객체 참조가 변하지 않아 리랜더링이 일어나지 않기 때문에 갱신 전, 후의 값을 직접 비교하기 위해서 Proxy 객체를 활용할 수 있다. Proxy는 객체 연산을 감지해서 핸들러를 적용할 수 있는 JS 내장 객체이다.

Valtio는 Proxy를 활용하여 상태 변화를 감지하는 라이브러리이다.

<br />

## Proxy를 활용한 변경 감지 및 불변 상태 생성하기

### 스냅 숏 ( Snapshot )

→ Proxy를 활용하여 변경 가능한 객체에서 생성된 변경 불가능한 객체

```jsx
// 객체 생성
import { proxy, snapshot } from 'valtio';

// 변경 가능한 객체
const state = proxy({ count: 0 });

// 스냅숏 생성, 변경 불가능한 객체 생성
const snap1 = snapshot(state);

++state.count;

const snap2 = snapshot(state);
```

- 이떄 state와 snap2가 가지는 값은 `{ count : 0 }` 으로 동일하지만 서로 다른 참조 값을 가진다.
- state로 부터 새로운 스냅숏 snap2 가 파생되었고, snap1과 snap2의 동등성을 비교하여 차이를 확인할 수 있다.

```jsx
// 중첩 객체인 경우
const state2 = proxy({
  obj1: { count: 0 },
  obj2: { count: 0 },
  ㄴ,
});

const snap3 = snapshot(state2);

++state.ojb1.count;

const snap4 = snapshot(state2);
```

- snap3와 snap4는 일단 서로 다른 참조를 가진다.
- snap3.obj1과 snap4.obj1은 갱신이 되었기 때문에 `snap3.obj1 !== snap4.obj1` 이다.
- snap3.obj2와 snap4.obj2는 속성 변경이 일어나지 않았기 때문에 `snap3.obj2===snap4.obj2` 이다.

→ 불변 상태를 자유롭게 적용할 수 있다. 따라서 필요한 속성에 대해서만 스냅숏을 생성해서 메모리 사용량도 최적화 할 수 있다.

<br />

### Proxy를 활용한 리랜더링 최적화

```jsx
import { proxy, useSnapshot } from 'valito';

// 객체를 인자로 받아 새로운 프락시 객체 반환
const state = proxy({
  count1: 0,
  count2: 0,
});

const Counter1 = () => {
  // valito에서는 useSnapshot의 반환값을 snap으로 설정하는 것이 관례.
  // snap 객체는 Object.freeze를 통해 동결되어 있어서 기술적으로 변경하지 못함.
  // useSnapshot이 속성을 추적하고 그 기반으로 리랜더링을 감지.
  const snap = useSnapshot(state);

  const increase = () => ++state.count;

  return (
    <>
      {snap.count1} <button onClick={increase}>+</button>
    </>
  );
};
```

<br />

## 작은 애플리케이션 만들어보기

```jsx
const TodoItem = ({
  id,
  title,
  done,
}: {
  id: string,
  title: string,
  done: boolean,
}) => {
  return (
    <div>
      <input type='checkbox' checked={done} onChange={() => toggleTodo(id)} />
      <span
        style={{
          textDecoration: done ? 'line-through' : 'none',
        }}
      >
        {title}
      </span>
      <button onClick={() => removeTodo(id)}>Delete</button>
    </div>
  );
};

const MemoedTodoItem = memo(TodoItem);

const TodoList = () => {
  const { todos } = useSnapshot(state);
  return (
    <div>
      {todos.map((todo) => (
        <MemoedTodoItem
          key={todo.id}
          id={todo.id}
          title={todo.title}
          done={todo.done}
        />
      ))}
    </div>
  );
};
```

→ TodoItem이 리랜더링 되면 TodoList까지 리랜더링 된다.

```jsx
const TodoItem = ({ id }: { id: string }) => {
  const todoState = state.todos.find((todo) => todo.id === id);
  if (!todoState) {
    throw new Error('invalid todo id');
  }
  const { title, done } = useSnapshot(todoState);
  return (
    <div>
      <input type='checkbox' checked={done} onChange={() => toggleTodo(id)} />
      <span
        style={{
          textDecoration: done ? 'line-through' : 'none',
        }}
      >
        {title}
      </span>
      <button onClick={() => removeTodo(id)}>Delete</button>
    </div>
  );
};
```

→ TodoItem에 id만 전달 받아 id에 해당하는 데이터를 직접 찾아 사용하는 방식

→ id 값이 변했을 때만 리랜더링 된다. 다른 요소에 의한 리랜더링 방지 가능.

→ 접근 방식을 다르게 바라봤을 뿐, 성능상 이점이 있지는 않다.

<br />

## 이 접근 방식의 장단점

Valito의 두가지 상태 업데이트 모델

- 불변 갱신

  - 요소를 변경하기가 번거롭다.

    ```jsx
    // 배열에서
    [ ...array.slice(0, index), ...array.slice(index + 1) ]

    // 중첩 객체에서
    {
    	...state,
    	a: {
    		...state.a,
    		b: {
    			...state.b,
    			c: {
    				...state.a.b.c,
    				text: 'hi',
    			}
    		}
    	}
    }
    ```

- 변경 가능한 갱신

  - JS 사용이 가능하다

    ```jsx
    // 배열에서
    array.splice(index, 1);

    // 중첩 객체에서
    stat.a.b.c.text = 'hi';
    ```

- 장점

  → Valtio의 변경 가능한 갱신을 활용하면 코드 양을 줄일 수 있다.

  ```jsx
  // Valtio
  const Component = () => {
  	const { count } = useSnapshot(state);
  	...
  }

  // Zustand
  const Component = () => {
  	const count = useStore(state => state.count);
  	...
  }
  ```

  ```jsx
  // Valtio
  const Component = ({ showText }) => {
    const snap = useSnapshot(state);

    return (
      <>
        {snap.count} {showText ? snap.text : ''}
      </>
    );
  };

  // Zustand
  const Component = ({ showText }) => {
    const count = useStore((state) => state.count);
    const text = useStore((state) => (showText ? state.text : ''));

    return (
      <>
        {count} {text}
      </>
    );
  };
  ```

- 단점
  - 예측성이 떨어진다
    → 랜더링 최적화가 훅 내부에서 처리되기 때문에 동작을 디버깅하기 어려움

<br />

## 🧐 Object.freeze(), 불변성

불변성(Immutability)란 말그대로 변하지 않는 것을 의미.

Object.freeze()는 객체를 readonly로 만들어 불변성을 부여하는 방법 중 하나.

```jsx
const obj = Object.freeze({
  count: 1,
});

obj.count = 2;

console.log(obj); // { count : 1 } : 변화 없음
```

하지만 중첩 객체에 대해서 불변성을 부여할 수 없다.

```jsx
const obj = Object.freeze({
  a: {
    b: 1,
  },
});

obj.a.b = 2;

console.log(obj.a); // { b : 2 } : 변화 없음
```

그 외 불변성을 부여하는 방식

- **`Immer.js`**: 불변 객체를 편리하게 생성할 수 있는 라이브러리. 기존 객체를 변경하는 것처럼 코드를 작성하되, 내부적으로 불변 객체를 생성해 줍니다.

  ```jsx
  javascript
  코드 복사
  import { produce } from 'immer';

  const state = { key: 'value', nested: { key: 'nestedValue' } };

  const newState = produce(state, (draft) => {
    draft.key = 'newValue';
    draft.nested.key = 'newNestedValue';
  });

  console.log(newState); // { key: 'newValue', nested: { key: 'newNestedValue' } }
  console.log(state);    // 원본 객체는 변경되지 않음

  ```

- **Immutable.js**: Facebook에서 제공하는 불변 데이터 구조 라이브러리. 깊은 객체 구조도 불변성을 자동으로 보장.
- TypeScript를 사용하면 객체의 프로퍼티를 **읽기 전용**으로 선언할 수 있음. 컴파일 타임에 불변성을 검사해 주기 때문에 더욱 안전함.

```tsx
typescript
코드 복사
type MyObject = {
  readonly key1: string;
  readonly key2: string;
};

const obj: MyObject = { key1: 'value1', key2: 'value2' };

// 오류 발생: Cannot assign to 'key1' because it is a read-only property
obj.key1 = 'newValue';

```

```js
// valtio/src/vanilla.ts
...
export type Snapshot<T> = T extends { $$valtioSnapshot: infer S }
  ? S
  : T extends SnapshotIgnore
    ? T
    : T extends object
      ? { readonly [K in keyof T]: Snapshot<T[K]> }
      : T

...

type ProxyState = readonly [
  target: object,
  ensureVersion: (nextCheckVersion?: number) => number,
  addListener: AddListener,
]

...

export function snapshot<T extends object>(proxyObject: T): Snapshot<T> {
  const proxyState = proxyStateMap.get(proxyObject as object)
  if (import.meta.env?.MODE !== 'production' && !proxyState) {
    console.warn('Please use proxy object')
  }
  const [target, ensureVersion] = proxyState as ProxyState
  return createSnapshot(target, ensureVersion()) as Snapshot<T>
}

export function ref<T extends object>(obj: T) {
  refSet.add(obj)
  return obj as T & { $$valtioSnapshot: T }
}

```

https://github.com/pmndrs/valtio/blob/main/src/vanilla.ts#L310

→ Valtio 내부에서도 불변성을 Object.freeze로 부여하기 보다, readonly 타입인 ProxyState를 통해서 불변성을 부여한다고 이해했습니다..! (Object.freeze 코드가 없음...)

→ 왜 Object.freeze를 사용하지 않고, readonly 타입을 통해 불변성을 주입했을까요?

<br />

→ 🤷🏻 실무에서 불변성을 부여하는 방식을 직접 구헌해서 사용하는 경우가 많은지 궁금합니다..!
