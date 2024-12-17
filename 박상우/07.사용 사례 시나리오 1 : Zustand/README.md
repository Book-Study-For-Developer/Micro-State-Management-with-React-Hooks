## 모듈 상태와 불변 상태 이해하기

- Zustand는 상태를 유지하는 store를 만드는데 사용되는 라이브러리이다. Zustand는 주로 모듈 상태를 위해 설계 됐고 모듈에서 sotre를 정의하고 내보낸다.
- Zustand는 상태 객체 속성을 갱신할 수 없는 불변 상태 모델을 기반으로 한다.
  → 객체 속성을 갱신할 수 없고, 전체 상태를 새로운 객체로 정의하여 상태 변화를 반영한다.
  → 속성 값을 비교할 필요가 없고, 객체의 동등성만 비교하여 변경 여부를 판단한다.

```tsx
// 기본 용법
import create from 'zustand';

export const store = create(() => ({ create : 0 }));

console.log(store.getState()) // { create : 0 }

console.log(store.setState({ create : 1 })
//또는
console.log(store.setState((prev) => ({ count : prev.count + 1 }))

console.log(store.getState()) // { create : 1 }

const state1 = store.getState()
state1.count += 1 // 이렇게 활용할 수 없다. (x)

// 부분 상태 갱신
const store = create(() => ({ name: 'Park', age: 20 });
store.setState({
	name: 'Kim',
})
console.log(store.getState()) // { name: 'Kim', age: 20 }

// subscribe를 통한 상태 변화에 따른 콜백 등록
store.subscribe(() => {
	console.log(1)
});
```

<br/>

## 리액트 훅을 이용한 리랜더링 최적화

Zustand는 리액트 내에서 해당 속성을 사용하지 않는 컴포넌트 까지 리랜더링되는 것을 방지하기 위해서 앞서 사용했던 create 함수는 훅으로 활용할 수 있는 store를 반환합니다.

```jsx
import create from 'zustand';

// 훅으로 활용하기 때문에 값의 이름을 use~로 변경
const useStore = create(() => ({ count : 1, text: 'hi' }))

// count, text 속성에 대해서 모두 랜더링 영향을 받는다.
const Component = () => {
	const { count, text } = useStore();

	...
}

// count 속성만 선택하여 활용함으로서 다른 값에 대한 리랜더링을 방지한다.
const Component = () => {
	const count = useStore((state) => state.count);

	...
}
```

### 수동 랜더링 최적화

→ 사용하지 않는 상태 값에 의해 리랜더링 되는 것을 방지하기 위해 선택자 함수가 반환하는 결과 값을 비교하는 방식

→ 선택자 함수가 매번 새로운 객체를 반환한다면 리랜더링이 반복적으로 발생

```jsx
const Component = () => {
	const [{ count }] = useStore((state) => [{ count : state.count }]);

	...
}
```

→ 선택자 기반 최적화 방식은 명시적으로 선택자 함수를 작성하기 때문에 동작 예측이 가능함

→ 하지만 객체 참조에 대한 이해를 바탕으로 반환 값을 결정할 수 있어야함

### 구조화된 데이터 처리하기

```tsx
import { memo, useState } from 'react';
import { create } from 'zustand';

type Todo = {
  id: number;
  title: string;
  done: boolean;
};

type StoreState = {
  todos: Todo[];
};

type StoreAction = {
  addTodo: (title: string) => void;
  removeTodo: (id: number) => void;
  toggleTodo: (id: number) => void;
};

type StoreType = StoreState & StoreAction;

let nextId = 0;

// 기존 객체(prev)를 변경하지 않고 새로운 객체와 배열을 생성한다. -> 불변 방식
const useStore = create<StoreType>((set) => ({
  todos: [],
  addTodo: (title) =>
    set((prev) => ({
      todos: [...prev.todos, { id: ++nextId, title, done: false }],
    })),
  removeTodo: (id) =>
    set((prev) => ({
      todos: prev.todos.filter((todo) => todo.id !== id),
    })),
  toggleTodo: (id) =>
    set((prev) => ({
      todos: prev.todos.map((todo) =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      ),
    })),
}));

const selectRemoveTodo = (state: StoreType) => state.removeTodo;
const selectToggleTodo = (state: StoreType) => state.toggleTodo;

const TodoItem = ({ todo }: { todo: Todo }) => {
  const removeTodo = useStore(selectRemoveTodo);
  const toggleTodo = useStore(selectToggleTodo);
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

// 불필요한 리랜더링을 방지하기 위해 memo 사용 TodoItem의 props가 변경될 때만 리랜더링된다.
const MemoedTodoItem = memo(TodoItem);

const selectTodos = (state: StoreType) => state.todos;

const TodoList = () => {
  const todos = useStore(selectTodos);
  return (
    <div>
      {todos.map((todo) => (
        <MemoedTodoItem key={todo.id} todo={todo} />
      ))}
    </div>
  );
};

const selectAddTodo = (state: StoreType) => state.addTodo;

const NewTodo = () => {
  const addTodo = useStore(selectAddTodo);
  const [text, setText] = useState('');
  const onClick = () => {
    addTodo(text);
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

<br/>

## 이 접근 방식과 라이브러리의 장단점

Zustand의 읽기와 쓰기 상태

- 읽기 : 리랜더링 최적화를 위한 선택자 함수 활용
- 쓰기 : 불변 상태 모델 기반

→ 결국 리액트가 객체 불변성을 기반으로 객체 참조 동등성을 통해 리랜더링하기 때문에 선택된 방식들이다.

→ 참조적으로 동일한 값을 반환하면 객체가 변경되지 않은 것으로 간주하는 것을 활용하여 리랜더링 최적화를 적용한다.

Zustand의 장단점

- 리액트와 동일한 모델을 적용하여 단순하고, 번들의 크기가 작다는 점에서 이점이 있음.
- 수동 랜더링 최적화를 활용하기 때문에, 객체 불변성에 대한 이해가 있어야하며, 선택자 함수를 활용하기 위해 별도의 보일러플레이트가 필요할 수 있음.
