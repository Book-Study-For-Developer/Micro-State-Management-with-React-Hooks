## Jotai

https://jotai.org/

> Jotai takes an atomic approach to global React state management.
>
> Build state by combining atoms and renders are automatically optimized based on atom dependency. This solves the extra re-render issue of React context, eliminates the need for memoization, and provides a similar developer experience to signals while maintaining a declarative programming model.

> Jotai는 전역 React 상태 관리에 원자적 접근 방식을 취합니다.
> 원자들을 결합하여 상태를 구축하고 원자 의존성을 기반으로 렌더링이 자동으로 최적화됩니다. 이는 React 컨텍스트의 추가 재렌더링 문제를 해결하고, 메모이제이션의 필요성을 제거하며, 선언적 프로그래밍 모델을 유지하면서 시그널과 유사한 개발자 경험을 제공합니다.

## 사용법 ?

```tsx
import { atom, useAtom } from "jotai";

// atom을 선언
const countAtom = atom(0);

const Counter1 = () => {
  // 해당 atom을 컴포넌트에 직접 사용
  const [count, setCount] = useAtom(countAtom);
  const inc = () => setCount((c) => c + 1);
  return (
    <>
      {count} <button onClick={inc}>+1</button>
    </>
  );
};

const Counter2 = () => {
  // 다른 컴포넌트에서 같은 atom을 사용하여 상태를 공유할 수 있음.
  const [count, setCount] = useAtom(countAtom);
  ...

}
```

즉 기존에 상태를 공유하기 위해 부모에 `Context`를 선언하고 `Provider`로 감싸는 불필요한 작업이 사라지게 된다. **atom만 선언한 뒤에 각 컴포넌트에서 useAtom을 통해 상태를 가져와 쓰면 되는 것**이다.

### 동적 아톰 생성

Jotai는 기본적으로 WeekMap 형태로 구성되어 있다.

> **WeakMap 이란?**

키가 약하게 참조되는 키/값 쌍의 컬렉션으로, **키는 반드시 객체**여야만 한다. 원시 값은 키가 될 수 없다. 보통 자바스크립트에서는 GC에 의해 객체들을 메모리에서 지워내지만, 자료구조에 속한 객체(Map, Set)의 경우 GC의 대상에서 제거되는데 이를 해소시켜주는 역할을 한다.

>

아톰으로 선언하게 되면 이는 리렌더링을 감지하는 단위가 된다.

```tsx
const firstNameAtom = atom("react");
const lastNameAtom = atom("hooks");
const ageAtom = atom(3);
```

이렇게 각각 아톰으로 선언해서 사용해도 되지만, 이렇게 할 경우 관리해야 할 아톰이 너무 많아질 수 있는 여지가 있다. (실무에서는 데이터가 객체 형태인 경우가 다반수이니.. )

즉, 객체인 atom을 만들어야 하는데 아래와 같이 하면 된다.

```tsx
const personAtom = atom((get) => ({
	firstName: get(firstNameAtom),
	lastName: get(lastNameAtom),
	age: get(ageAtom),
})
```

파생된 아톰을 만들어서 각각의 아톰의 하나라도 변하게 되면 감지하여 갱신하게 된다. ⇒ **의존성(종속성) 추적(dependency tracking)**이라고 불림

> **의존성 추적이란?**

책에서는 삼항 연산자를 예시로 의존성이 동적으로 바뀌는 예시를 들었는데, 쉽게 말하면 의존성은 동적으로 정해지면 정해진 의존성에 따라 자동으로 추적하는 것을 말한다.

**-** jotai내에서 테스트 코드로 의존성에 따른 변경을 확인하는 것으로 보인다.https://github.com/pmndrs/jotai/blob/main/tests/vanilla/dependency.test.tsx

>

❓: 이렇게 관리하는 건 결국엔 atom을 한번 선언하고 파생된 atom을 관리하는 것인데 jotai를 한번도 써보지 않았지만 귀찮은 느낌이 든다.

`personAtom`은 `ageAtom`이 변할 때 리렌더링이 일어날까요? ⇒ 네

- 불필요한 리렌더링이 일어나게 되므로 age는 불리해야 한다.

```tsx
const fullNameAtom = atom((get) => ({
	firstName: get(firstNameAtom),
	lastName: get(lastNameAtom),
})
```

이렇게 **작은 아톰**에서부터 **큰 아톰**으로 만드는 방식을 **상향식(bottom-up) 접근법**이라고 한다.
추후 스펙 변경이 일어나면 사용하게 될 아톰만 추가하여 최적화를 해야 한다. (당연하게 수동임..)

### useStoreSelector의 문제점 해결하기

useStoreSelector를 사용하게 되면 객체를 가져오는 selector로 가져오게 되더라도 함수가 다시 평가되어 새로운 객체를 반환하기 때문에 리렌더링을 유발하게 된다. (p154)

아톰을 사용하면 **리렌더링을 감지하는 단위로 선언하기 때문에 제어가 간단**해진다.

```tsx
const count1Atom = atom(0);
const count2Atom = atom(0);

const Counter ({ countAtom }) => {
	const [count, setCount] = useAtom(countAtom);
	const inc = () => setCount((c) => c + 1);
	return <>{count} <button onClick={inc}>+1</button></>;
}

const totalAtom = atom((get) => get(count1Atom) + get(count2Atom));
```

`totalAtom`의 경우 의존성 추적에 의해 count1Atom과 count2Atom이 변할 때만 새롭게 값을 갱신하게 된다.

## Jotai가 Atom을 저장하는 방식

Jotai는 결국 useContext를 사용한다고 하였는데 이는 여기에 들어있음https://github.com/pmndrs/jotai/blob/main/src/react/Provider.ts#L16-L18

### **그렇다면 react 의 context API보다 나은점이 무엇인가?**

일단 context API가 지원하는 모든 기능을 커버하고 있고, atom 자체에 값을 가지고 있지 않아 재사용성이 높다.

그래서 Provider마다 각각 다른 값을 가지게 세팅할 수 있는 것이다.

```tsx
const countAtom = atom(0);

const App = () => {
  return (
    <>
      <Provider></Provider>
      <Provider></Provider>
      <Provider></Provider>
      ...
    </>
  );
};
```

**❓: countAtom 자체에 값을 가지고 있지 않다는 말이 무엇일까?**

- 위의 코드를 보면 `atom(0)`으로 0이라는 값을 주었다. 그런데 책에서는 값이 없다고 설명하는데 이 말을 다시 생각해보면 **결국 각 컨텍스트마다 새롭게 상태를 정의(기존 리액트에서 하는 방식)하는 것이 아니라 하나의 atom으로 각각의 Provider마다 독립적인 값으로 사용이 가능하기 때문에** 없다고 하고 있다.

## Atoms-in-Atom 패턴 살펴보기

Zustand에서 했듯이 todo의 예제를 Jotai를 사용해서 구현하는 코드를 보여준다. 마찬가지로 예시 코드를 살펴보는게 더 나아서 따로 작성하진 않았다.

https://github.com/wikibook/msmrh/blob/main/chapter08/04_todo_app_single_atom/src/App.tsx

**코드의 문제점**

1.  단일 요소를 변경하기 위해 todos 배열 전체를 갱신해야 한다.

    ```tsx
    const [text, setText] = useState("");
    // 전체 todo를 갱신해야 함.
    const onClick = () => {
      setTodos((prev) => [...prev, { id: nanoid(), title: text, done: false }]);
      setText("");
    };
    ```

2.  반복 렌더링 key를 id값으로 사용해야 하는데 사용하지 않는 것이 좋다
    (❓ : 왜 id를 사용하지 않는 것이 좋다고 하는 거지..? ⇒ 뒤를 읽어보니 결국 atom의 특성상 유니크하게 만들 방법이 있으니 불필요하게 id까지 관리하지 말자는 의미인 것 같다.)
        ```tsx
        {todos.map((todo) => (
           <MemoedTodoItem
              key={todo.id}
              todo={todo}
              removeTodo={removeTodo}
              toggleTodo={toggleTodo}
           />
         ))}
        ```

**패턴 사용해서 해결하기**

```tsx
type Todo = {
  title: string;
  done: boolean;
};

type TodoAtom = PrimitiveAtom<Todo>;
const todoAtomsAtom = atom<TodoAtom[]>([]);

const MemoedTodoItem = memo(TodoItem);

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
      // todo를 갱신하더라도 todoAtomsAtom는 변하지 않아 리렌더링이 일어나지
      않는다.
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => setTodo((prev) => ({ ...prev, done: !prev.done }))}
      />
      <span
        style={{
          textDecoration: todo.done ? "line-through" : "none",
        }}
      >
        {todo.title}
      </span>
      <button onClick={() => remove(todoAtom)}>Delete</button>
    </div>
  );
};

const TodoList = () => {
  // Atom안에 Atom인 값을 가져온다.
  const [todoAtoms, setTodoAtoms] = useAtom(todoAtomsAtom);

  const remove = useCallback(
    (todoAtom: TodoAtom) =>
      setTodoAtoms((prev) => prev.filter((item) => item !== todoAtom)),
    [setTodoAtoms]
  );

  return (
    <div>
      {todoAtoms.map((todoAtom) => (
        // 그렇게 하면 key에 id가 아닌 todoAtom(uniq 한 값)을 그대로 사용할 수 있게 된다.
        // 왜냐하면 Atom을 string으로 평가하면 유니크하게 된다고 함
        // @see https://jotai.org/docs/guides/atoms-in-atom#storing-an-array-of-atom-configs-in-atom
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
  //
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
```

❓: 간단한 예시인 투두로도 아톰안에 아톰 패턴을 써서 하는게 좀 복잡한 느낌인데, 어러운 배열/객체를 다룰때는 더 힘들어질 느낌이 든다..

책에선 Jotai의 고급 사용법에 대해 설명하는데, 책과 크게 연관이 없는 것 같아 적지 않았다.
