useState, useReducer와 함께 ‘아톰’을 기반으로 상태를 관리하는 라이브러리이다. 컨텍스트 및 구독 패턴을 기반으로 한다.

## Jotai 이해하기

Jotai를 활용했을 때의 두가지 이점

1. 구문 단순성
2. 동적 아톰 생성

### 1. 구문 단순성

```jsx
// jotai를 활용하는 경우
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);

const Counter1 = () => {
	const [ count, setCount ] = useAtom(countAtom);
	const inc = () => setCount((c) => c + 1);

	...
}
```

```jsx
// Context를 활용하는 경우
const countContext = createContext();

const CounterProvider = ({ children }) => (
  <CounterContext.Provider value={useState('')}>
    {children}
  </CounterContext.Provider>
);

const App = () => {
  <TextProvider>...</TextProvider>;
};
```

이전에 지역 상태를 공유하기 위해 활용하는 패턴 중 ‘상위 컴포넌트로 끌어올리기’ 패턴과 그 상태를 Context로 주입하는 과정을 알아봤었다. Jotai를 활용했을 때는, 활용하는 부분에서는 useState가 useAtom으로 변경된 것 이외에 변화가 없지만, 그 상위 컴포넌트에서 만드는 Context와 컴포넌트를 감싸는 Provider를 정의할 필요가 없다. 그리고 Provider를 만들 필요가 없기 때문에 여러 컨텍스트를 사용할 때 Provider가 중첩되는 현상도 당연히 없다.

### 2. 동적 아톰 생성

Jotai 스토어는 기본적으로 아톰 구성 객체와 아톰 값으로 구성된 WeakMap 객체이다. 아톰 구성 객체는 atom 함수로 생성된다. 아톰 값은 useAtom 훅이 반환하는 값이다. Jotai 내에서 구독은 아톰 기반으므로 useAtom 훅이 store의 특정 값을 구독하고 있음을 의미한다. 해당 원리로 Jotai는 리랜더링을 피하고 있다.

<br />

## 랜더링 최적화

```jsx
const pearsonStore = createStore({
	firstName: 'React',
	lastName: 'Hooks',
});

const selectFirstName = (state) => state.firstName;
const selectLastName = (state) => state.lastName;

const PersonComponent = () => {
	const firstName = useStoreSelector(store, selectFirstTime);
	const lastName = useStoreSelector(store selectLastName);

	return <>{firstName} {lastName}</>
}
```

지난 `구독을 통한 모듈 상태 공유` 에서 만든 예시들은 store와 선택자를 접근하는 방식에서 하향식(top-down) 접근법을 사용

```jsx
const firstNameAtom = atom('React');
const lastNameAtom = atom('Hooks');

const PersonComponent = () => {
  const [firstName] = useAtom(firstNameAtom);
  const [lastName] = useAtom(lastNameAtom);

  return (
    <>
      {firstName} {lastName}
    </>
  );
};
```

→ Jotai에서는 작은 단위로 Atom을 만들어 리랜더링을 제어할 수 있지만, 너무 많은 atom을 만들어야한다는 단 점이 있음. 그래서 아래와 같이 다른 아톰을 만들 수 있는 파생 아톰이라는 개념을 활용

```jsx
const personAtom = atom((get) => ({
	firstName: get(firstNameAtom),
	lastName: get(lastNameAtom),
})
```

이 파생 아톰은 속성값 중에 하나가 갱신될 때마다 리랜더링 됨. 이것을 의존성 추적(Dependency Tracking)이라고 한다.

> 의존성 추적은 동적으로 판단이 가능하다. get ⇒ get(a) ? Get(b) : get(c) 이렇게 작성하여 a 여부에 따라 b에 의존성을 가질 것인지 c에 의존성을 가질 것인지 판단이 가능하다.

파생 아톰을 활용하는 컴포넌트는 파생 아톰 내 모든 속성에 의해 리랜더링이 발생하기 때문에 필요한 값만 포함하도록 해야한다.

→ 작은 아톰을 통해 큰 파생 아톰을 만들어내내는 것을 상향식(Bottom-up) 접근이라고 한다. 필요한 아톰만 결합하여 리랜더링을 최적화 할 수 있다.

선택자 함수를 활용한 리랜더링 최적화

```jsx
const count1Atom = atom(0);
const count2Atom = atom(0);

const totalAtom = atom((get) => get(count1Atom) + get(count2Atom));
```

T totalAtom 과 같은 파생 아톰을 생성할 떄 read 함수를 활용하여 생성힌다. 파생 아톰의 값은 read의 결과 값이기 때문에 의존성을 가진 아톰이 갱신될 때만 값을 갱신한다. 읽기 전용으로만 활용할 수 있으며, 설정을 할 수 없다.

<br />

## Jotai가 아톰 값을 저장하는 방식 이해하기

```jsx
// Provider를 활용하여 컴포넌트 레벨에서 store 생성이 가능하다.
import { atom, useAtom, Provider } from 'jotai'

// atom은 아톰 동작을 포함하는 몇가지 속성을 가진 객체
// 실제로 atom이 해당 값을 가지는 것이 아니고 별도의 store에 저장된다.
// store는 아톰 구성 객체를 키로가지고 WeakMap 객체를 값으로 가지는 객체로 되어 있다.
const countAtom = atom(0);

const Counter = ({ countAtom }) => {
	const [ count, setCount ] = useAtom(countAtom);
	const inc = () => setCount((c) => c + 1);
	return (
		<>
			{count} <button onClick={inc}>+1</button>
		</>
	)
}

const App = () => {
	return (
		<>
			<Provider>
				<p>first</p>
				<Counter />
				...
			<Provider/>
			<Provider>
				<p>second</p>
				<Counter />
				...
			<Provider/>
		</>
	)
}
```

서로 다른 Provider 컴포넌트는 서로 다른 스토어를 가진다 . CountAtom은 값을 가지고 있는 것이 아니기 때문에 여러 provider 내에서 재활용 가능하다.

<br />

## 배열 구조 추가하기

리액트에서 배열의 값을 삭제하고 순서를 변경하기가 상당히 까다롭지만 아톰 속의 아톰이라 불리는 패턴을 통해 Jotai에서 배열을 수정, 삭제한다.

```tsx
import { memo, useCallback, useState } from 'react';
import { atom, useAtom } from 'jotai';
import { nanoid } from 'nanoid';

type Todo = {
  id: string;
  title: string;
  done: boolean;
};

const todosAtom = atom<Todo[]>([]);

const TodoItem = ({
  todo,
  removeTodo,
  toggleTodo,
}: {
  todo: Todo;
  removeTodo: (id: string) => void;
  toggleTodo: (id: string) => void;
}) => {
  return (
    <div>
      <input
        type='checkbox'
        checked={todo.done}
        onChange={() => toggleTodo(todo.id)}
      />
      <span
        style={{
          textDecoration: todo.done ? 'line-through' : 'none',
        }}
      >
        {todo.title}
      </span>
      <button onClick={() => removeTodo(todo.id)}>Delete</button>
    </div>
  );
};

const MemoedTodoItem = memo(TodoItem);

const TodoList = () => {
  const [todos, setTodos] = useAtom(todosAtom);

  // useCallback을 활용하여 함수 재생성 방지
  const removeTodo = useCallback(
    (id: string) => setTodos((prev) => prev.filter((item) => item.id !== id)),
    [setTodos]
  );
  const toggleTodo = useCallback(
    (id: string) =>
      setTodos((prev) =>
        prev.map((item) =>
          item.id === id ? { ...item, done: !item.done } : item
        )
      ),
    [setTodos]
  );
  return (
    <div>
      {todos.map((todo) => (
        <MemoedTodoItem
          key={todo.id}
          todo={todo}
          removeTodo={removeTodo}
          toggleTodo={toggleTodo}
        />
      ))}
    </div>
  );
};

const NewTodo = () => {
  const [, setTodos] = useAtom(todosAtom);
  const [text, setText] = useState('');
  const onClick = () => {
    setTodos((prev) => [...prev, { id: nanoid(), title: text, done: false }]);
    setText('');
  };
  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={onClick} disabled={!text}>
        Add
      </button>
    </div>
  );
};

const App = () => (
  <>
    <TodoList />
    <NewTodo />
  </>
);

export default App;
```

위 예제에서 우려되는 두가지 문제점

1. 단일 요소 변경을 위해 todos 전체 배열이 갱신되어야한다는 점
2. id 값을 사용하면 안된다는 점

   → 🧐 why..? 일관되게 생성되지 않아서…?

### Atoms-in-Atom 패턴

```tsx
import { memo, useCallback, useState } from 'react';
import { PrimitiveAtom, atom, useAtom } from 'jotai';

type Todo = {
  title: string;
  done: boolean;
};

// Jotai를 활용하여 제네릭 타입을 활용해 배열 내 요소의 타입을 생성
type TodoAtom = PrimitiveAtom<Todo>;

// Atoms-in-Atom
const todoAtomsAtom = atom<TodoAtom[]>([]);

const TodoItem = ({
  todoAtom,
  remove,
}: {
  todoAtom: TodoAtom;
  remove: (todoAtom: TodoAtom) => void;
}) => {
  const [todo, setTodo] = useAtom(todoAtom);
  return (
    <div>
      <input
        type='checkbox'
        checked={todo.done}
        onChange={() => setTodo((prev) => ({ ...prev, done: !prev.done }))}
      />
      <span
        style={{
          textDecoration: todo.done ? 'line-through' : 'none',
        }}
      >
        {todo.title}
      </span>
      <button onClick={() => remove(todoAtom)}>Delete</button>
    </div>
  );
};

const MemoedTodoItem = memo(TodoItem);

const TodoList = () => {
  const [todoAtoms, setTodoAtoms] = useAtom(todoAtomsAtom);
  const remove = useCallback(
    (todoAtom: TodoAtom) =>
      setTodoAtoms((prev) => prev.filter((item) => item !== todoAtom)),
    [setTodoAtoms]
  );
  return (
    <div>
      {todoAtoms.map((todoAtom) => {
        // atom이 문자열로 평가될 때 유일한 식별자(UID)를 반환
        // 'atom2', 'atom3' ... 이런 식으로 텍스트로 값이 저장되어 있음
        console.log(`${todoAtom}`);
        return (
          <MemoedTodoItem
            key={`${todoAtom}`}
            todoAtom={todoAtom}
            remove={remove}
          />
        );
      })}
    </div>
  );
};

const NewTodo = () => {
  const [, setTodoAtoms] = useAtom(todoAtomsAtom);
  const [text, setText] = useState('');
  const onClick = () => {
    setTodoAtoms((prev) => [...prev, atom<Todo>({ title: text, done: false })]);
    setText('');
  };
  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={onClick} disabled={!text}>
        Add
      </button>
    </div>
  );
};

const App = () => (
  <>
    <TodoList />
    <NewTodo />
  </>
);

export default App;
```

<img width="1118" alt="스크린샷 2024-12-17 오전 12 19 58" src="https://github.com/user-attachments/assets/a471febc-4d72-40b8-8681-578e57182738" />



#### Atoms-in-Atom을 활용했을 때의 차이점

- 배열 아톰은 아톰이 요소인 배열을 보관하는데 사용된다.
- 배열에 새로운 요소를 추가하려면 새로운 아톰을 생성해서 추가해야 한다.
- 아톰 구성은 무자열로 평가할 수 있으며 UID를 반환한다.
- 요소를 렌더링하는 컴포넌트는 각 컴포넌트의 아톰 요소를 사용한다.
  → 배열 요소가 갱신되더라도 전체 데이터에 해당하는 아톰(todoAtomsAtom)은 변경되지 않는다.

<br />

## 🧐 WeakMap

WeakMap 객체는 키가 약하게 참조되는 키/값 쌍의 컬렉션으로, 키는 반드시 객체여야만 한다. 원시 값은 키가 될 수 없다. 만약 키를 원시 값으로 추가하면 Uncaught TypeError: Invalid value used as weak map key라는 에러가 발생한다.
그리고 WeakMap은 가지고 있는 요소 전체를 반복 구문으로 탐색할 방법이 없다. 이것은 제약 사항처럼 보일 수 있지만 거기에는 그럴만한 이유가 있다. 실제, WeakMap의 독특한 특징은 키로 사용된 객체에 대한 유일한 참조가 WeakMap 내에만 남아 있을 경우, 이 객체를 가비지 컬렉트(garbage collect)할 수 있다.

```js
const weakMap = new WeakMap();
const objKey = { name: 'keyObject' };

weakMap.set(objKey, 'value');
console.log(weakMap.get(objKey)); // "value"

// 원시 값은 사용할 수 없음
weakMap.set('key', 'value'); // TypeError
```

https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/WeakMap

<br />

## 🧐 jotai에서는 왜 nanoid를 썼을까

zustand에 이어 jotai 부분을 읽다가 Todo 시 zustand는 전역 변수로, jotai는 nanoid로, 각각 id를 다르게 적용하는 것을 보고 왜 다르게 적용하는 생각해보았습니다.

당연한 이야기일 수 있지만 두 방식이 전역 상태를 관리하고, 상태를 변화시키는 함수의 위치가 서로 다르기 때문입니다.

zustand의 경우 상태, 상태 변화 로직이 모두 store 내부에 위치합니다.

반면에 jotai는 state가 필요한 컴포넌트 내부에 useState 처럼 상태를 호출(?)하고, 상태를 변화시키는 함수가 있습니다.

따라서 zustand는 어느 곳에서 상태 변화가 발생하든 zustand store가 선언된 곳에서 상태가 갱신되기 때문에 해당 스코프에서 선언한 변수를 활용해도 id로서 활용이 가능했던 것이고,

Jotai의 경우 상태 변화가 어디서 발생할지 예측하기 어렵고, 컴포넌트마다 서로 다른 시점의 상태를 가지고 있기 때문에 전역 변수 보다 항상 구분 가능한 값을 보장할 수 있는 nanoid를 활용하는 것으로 이해했습니다.
