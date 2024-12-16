## 사용 사례 시나리오 2: Jotai

Jotai는 Zustand와 마찬가지로 불변 상태 모델이지만, **컴포넌트 상태**를 사용한다는 점이 다르다. 그리고 useState, useReducer와 함께 상태의 작은 조각인 **아톰**이라고 불리는 것을 모델로 삼는다.

### Jotai 이해하기

기존에 배웠던 Context를 활용하여 상태를 공유하는 방법을 다시 알아보자.

```tsx
const Counter1 = () => {
  const [count, setCount] = useState(0);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

const Counter2 = () => {
  const [count, setCount] = useState(0);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

const App = () => (
  <>
    <div>
      <Counter1 />
    </div>
    <div>
      <Counter2 />
    </div>
  </>
);
```

Counter1와 Counter2는 고유한 지역 상태가 있기 때문에 컴포넌트에서 보여주는 숫자는 서로 격리되어 있다. 두 컴포넌트가 하나의 카운트 상태를 공유하게 하려면 상태를 상위 컴포넌트로 끌어올리고 컨텍스트를 사용해서 전달할 수 있다.

```tsx
const CountContext = createContext();

const CountProvider = ({ children }) => (
  <CountContext.Provider value={useState(0)}>{children}</CountContext.Provider>
);

const Counter1 = () => {
  const [count, setCount] = useContext(CountContext);
  // inc 함수 생략
  // return 생략
};

// const Counter2 = ...

const App = () => (
  <CountProvider>
    <div>
      <Counter1 />
    </div>
    <div>
      <Counter2 />
    </div>
  </CountProvider>
);
```

### 구문 단순성

컨텍스트 사용과 비교했을 때 Jotai를 쓰면 **구문 단순성**을 가져올 수 있다.

```tsx
// atom은 작은 상태 조각이면서 리렌더링을 감지하는 최소 단위
import { atom, useAtom } from "jotai";

// 아톰 생성
const countAtom = atom(0);

const Counter1 = () => {
  const [count, setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

const Counter2 = () => {
  const [count, setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

// 공급자(provider)를 사용하지 않아도 상태 공유가 가능하다.
const App = () => (
  <>
    <div>
      <Counter1 />
    </div>
    <div>
      <Counter2 />
    </div>
  </>
);
```

> [!WARNING]
> 물론, 서로 다른 하위 트리에 대해 각각 다른 값을 제공해야 하는 경우는 공급자를 선택적으로 사용해야 한다.

### 동적 아톰 생성

📌 아톰은 리액트 컴포넌트 생명 주기에서 생성되거나 소멸될 수 있다. 반면, 다중 컨텍스트 접근 방식에서는 새로운 상태를 추가한다는 것은 새로운 공급자 컴포넌트를 추가한다는 것을 의미하기 때문에 리액트 컴포넌트 생명 주기에서 생성되거나 소멸되는 것이 불가능하다..!

### 렌더링 최적화

```tsx
const firstNameAtom = atom("React");
const lastNameAtom = atom("Hooks");
const ageAtom = atom(3);

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

PersonComponent는 ageAtom과 관련이 없으므로 ageAtom의 값이 변경되더라도 리렌더링되지 않는다.

아톰은 원시 타입만큼 작게 만드는 것이 가능하지만, 조작해야 할 아톰이 너무 많아지는 것을 방지하기 위해 파생 아톰을 만들 수 있다.

```tsx
const personAtom = atom((get) => {
  firstName: get(firstNameAtom),
  lastName: get(lastNameAtom),
  age: get(ageAtom),
});
```

다른 아톰을 참조하고 그 값을 가져올 수 있는 get이라는 인수를 받는 read 함수를 쓸 수 있다.

```tsx
// 그런데 아래 컴포넌트는 ageAtom의 값이 변경되면 리렌더링된다...
const PersonComponent = () => {
  const person = useAtom(personAtom);
  return (
    <>
      {person.firstName} {person.lastName}
    </>
  );
};
```

그래서, 해결 방안으로 firstNameAtom과 lastNameAtom만 취급하는 파생 아톰을 만들어서 사용하면 된다.

이를 **상향식(bottom-up) 접근법**이라고 한다. 작은 아톰을 만들고 이를 결합해 더 큰 아톰을 만든다. 컴포넌트에 사용될 아톰만 추가해서 리렌더링을 최적화할 수 있다.

### Jotai가 아톰 값을 저장하는 방식 이해하기

`const countAtom = atom(0);`에서 countAtom은 아톰 동작을 나타내는 몇 가지 속성을 가진 객체다. 중요한 점은, countAtom과 같은 아톰 구성이 직접 해당 값(0)을 가지지 않는다는 것이다. 아톰 값을 저장하는 store가 따로 있다. store에는 키가 아톰 구성 객체이고 값이 아톰 값인 WeakMap 객체가 있다.

```tsx
import { atom, useAtom, Provider } from "jotai";

const Counter = ({ countAtom }) => {
  const [count, setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

const App = () => (
  <>
    <Provider>
      <h1>First Provider</h1>
      <div>
        <Counter />
      </div>
      <div>
        <Counter />
      </div>
    </Provider>
    <Provider>
      <h1>Second Provider</h1>
      <div>
        <Counter />
      </div>
      <div>
        <Counter />
      </div>
    </Provider>
  </>
);
```

신기한 점은, Jotai에서 제공하는 Provider 컴포넌트는 서로 다른 스토어를 사용하기 때문에 첫 Provider 내부에서 공유하는 countAtom과 두 번째 Provider 내부에서 공유하는 countAtom은 서로 다르다.

🔑 **여기서 중요한 것은 countAtom 자체가 값을 갖고 있지 않다는 것이다.** 따라서, countAtom은 여러 Provider 컴포넌트에 재사용할 수 있다. 이것이 모듈 상태와 다른 점이다.

### 배열 구조 추가하기 - Atoms-in-Atom

```tsx
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
        type="checkbox"
        checked={todo.done}
        onChange={() => toggleTodo(todo.id)}
      />
      <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
        {todo.title}
      </span>
      <button onClick={() => removeTodo(todo.id)}>Delete</button>
    </div>
  );
};

// todo, removeTodo, toggleTodo를 변경하는 것이 아니면 리렌더링이 되지 않도록 메모이제이션 적용
const MemoedTodoItem = memo(TodoItem);

const TodoList = () => {
  const [todos, setTodos] = useAtom(todosAtom);

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

// 새로운 todos를 갱신하는 버튼을 가진 컴포넌트
const NewTodo = () => {
  const [, setTodos] = useAtom(todosAtom);
  const [text, setText] = useState("");
  const onClick = () => {
    setTodos((prev) => [...prev, { id: nanoid(), title: text, done: false }]);
    setText("");
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
```

하지만 이러한 코드를 작성하면서 예견되는 우려사항이 두 가지 있다.

- 단일 요소를 변경하기 위해 todos 배열 전체를 갱신해야 한다. MemoedTodoItem 덕분에 특정 요소가 변경되지 않는 한 리렌더링되지 않지만, 이상적으로는 특정 MemoedTodoItem 컴포넌트가 리렌더링되도록 감지하는 것이 좋다.
- 요소의 id 값이 문제다. id 값은 주로 map에서 key로 사용되는데 id는 사용하지 않는 것이 좋다.

이 두 문제를 해결하기 위해서 아톰 구성을 다른 아톰 값에 넣는 Atoms-in-Atom 패턴을 쓸 수 있다.

```tsx
type Todo = {
  title: string;
  done: boolean;
};

type TodoAtom = PrimitiveAtom<Todo>;

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
        type="checkbox"
        checked={todo.done}
        onChange={() => setTodo((prev) => ({ ...prev, done: !prev.done }))}
      />
      <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
        {todo.title}
      </span>
      <button onClick={() => remove(todoAtom)}>Delete</button>
    </div>
  );
};

const MemoedTodoItem = memo(TodoItem);

const TodoList = () => {
  const [todoAtoms, setTodoAtoms] = useAtom(todosAtomsAtom);

  const remove = useCallback(
    (todoAtom: TodoAtom) =>
      setTodoAtoms((prev) => prev.filter((item) => item !== todoAtom)),
    [setTodosAtoms]
  );

  return (
    <div>
      {todoAtoms.map((todoAtom) => (
        <MemoedTodoItem
          key={`${todoAtom}`}
          todoAtom={todoAtom}
          remove={remove}
        />
      ))}
    </div>
  );
};

const NewTodo = () => {
  const [, setTodoAtoms] = useAtom(todoAtomsAtom);
  const [text, setText] = useState("");
  const onClick = () => {
    setTodoAtoms((prev) => [...prev, atom<Todo>({ title: text, done: false })]);
    setText("");
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
```

TodoList에서 map을 사용할 때 key는 문자열로 변환한 todoAtom 구성으로 지정한다. 📌 아톰 구성은 문자열로 평가될 때 유일한 식별자(UID)를 반환하기 때문에 id를 따로 관리할 필요가 없다!

또한, TodoList 컴포넌트 동작은 이전과 다르게, Atoms-in-Atom을 다루기 때문에 배열 요소 하나가 toggleTodo를 통해 갱신되어도 todoAtomsAtom은 변경되지 않아서 리렌더링을 줄일 수 있다.

결국, Atoms-in-Atom 패턴을 사용했을 때의 차이점은 다음과 같다.

- 배열 아톰은 아톰이 요소인 배열을 보관하는 데 사용한다.
- 배열에 새로운 요소를 추가하려면 새로운 아톰을 생성해서 추가해야 한다.
- 아톰 구성은 문자열로 평가할 수 있고 UID를 반환한다.
- 요소를 렌더링하는 컴포넌트는 각 컴포넌트에서 아톰 요소를 사용한다. 요소의 값을 쉽게 변경할 수 있고 리렌더링을 피할 수 있다.
