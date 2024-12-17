## 프락시를 활용한 변경 감지 및 불변 상태 생성하기

```tsx
const state = proxy({ count: 0 })

const snap1 = snapshot(state) // { count: 0 }
++state.count
const snap2 = snapshot(state) // { count: 1 }
```

- Valtio는 프락시를 사용해 변경 가능한 객체에서 변경 불가능한 객체를 생성 → **스냅숏(snapshot)**
  - `state`: 프락시로 감싼 변경 가능한 객체
  - `snap1`: `Object.freeze`로 동결되어 변경 불가능한 객체
- 중첩된 객체에 대해서는 변경되지 않았다면 참조 유지 → 스냅숏 최적화
- 스냅숏 최적화
  - 참조가 동일하다? → _메모리를 공유한다_
  - 필요한 경우에만 스냅숏을 생성해서 메모리 사용향 최적화
  - 단, Valtio의 최적화는 **이전 스냅숏에 대한 캐싱을 기반**

## 프락시를 활용한 리렌더링 최적화

```tsx
import { proxy, useSnapshot } from 'valtio'

// prox로 state 객체 생성
const state = proxy({
  count1: 0,
  count2: 0,
})

const Counter1 = () => {
  // useSnapshot 훅으로 추적 정보 감지 → 컴포넌트가 어떤 속성에 접근했는지?
  // Object.freeze로 동결된 snap 객체(변수명은 컨벤션)
  const snap = useSnapshot(state)
  // 스냅숏 객체의 count1 속성에 접근
  const inc = () => ++state.count1
  return (
    <>
      {snap.count1} <button onClick={inc}>+1</button>
    </>
  )
}

const Counter2 = () => {
  const snap = useSnapshot(state)
  // 스냅숏 객체의 count2 속성에 접근
  const inc = () => ++state.count2
  return (
    <>
      {snap.count2} <button onClick={inc}>+1</button>
    </>
  )
}

const App = () => (
  <>
    <div>
      <Counter1 />
    </div>
    <div>
      <Counter2 />
    </div>
  </>
)

export default App
```

## Valtio를 애플리케이션에 적용하기

- `state` 객체를 일반적인 자바스크립트 객체처럼 관리
  - 일반적인 자바스크립트? mutation 허용 → proxy로 감싸져 있기 때문

```tsx
const createTodo = (title: string) => {
  state.todos.push({
    id: nanoid(),
    title,
    done: false,
  })
}

const removeTodo = (id: string) => {
  const index = state.todos.findIndex(item => item.id === id)
  state.todos.splice(index, 1)
}

const toggleTodo = (id: string) => {
  const index = state.todos.findIndex(item => item.id === id)
  state.todos[index].done = !state.todos[index].done
}
```

- `memo`로 컴포넌트를 메모이제이션할 경우, props를 하나의 객체 아닌 분리해서 전달이 필요 → 상태 사용 추적은 속성 접근을 감지하기 때문에 메모된 컴포넌트에 객체를 전달하면 속성 접근을 생략

> 최근에 해결된 것 같다!
>
> - [Using React.memo with object props may result in unexpected behavior (v1 only)](https://valtio.dev/docs/how-tos/some-gotchas?utm_source=chatgpt.com)

> 어떻게 해결했는지 찾아보려고 했으나.. 아직 다 파악을 못 함 😭
>
> - [#884 dai-shi comment](https://github.com/pmndrs/valtio/discussions/884#discussioncomment-9064588)
> - [#866](https://github.com/pmndrs/valtio/pull/866)

```tsx
const MemoedTodoItem = memo(TodoItem)

const TodoList = () => {
  const { todos } = useSnapshot(state)
  return (
    <div>
      {todos.map(todo => (
        <MemoedTodoItem key={todo.id} id={todo.id} title={todo.title} done={todo.done} />
      ))}
    </div>
  )
}
```

#### TodoItem에서 useSnapshot을 사용하여 리렌더링 최적화하기

- id 속성을 기반으로 todoState 찾기 → todoState으로 useSnapshot title, done 가져오기 → TodoList가 id만 관련되게 변경하여 리렌더링 최적화 가능

```tsx
const TodoItem = ({ id }: { id: string }) => {
  const todoState = state.todos.find(todo => todo.id === id)
  if (!todoState) {
    throw new Error('invalid todo id')
  }

  const { title, done } = useSnapshot(todoState)

  return (
    <div>
      <input type="checkbox" checked={done} onChange={() => todggleTodo(id)} />
      <span style={{ textDecoration: done ? 'line-throungh' : 'none' }}>{title}</span>
      <button onClick={() => removeTodo(id)}>Delete</button>
    </div>
  )
}

const MemoedTodoItem = memo(TodoItem)
```

```tsx
const TodoList = () => {
  const { todos } = useSnapshot(state)
  const todoIds = todos.map(todo => todo.id)

  return (
    <div>
      {todos.map(todoId => (
        <MemoedTodoItem key={todoId} id={todoId} />
      ))}
    </div>
  )
}
```

## 프락시를 활용한 접근 방식의 장단점

- Valtio의 멘탈 모델: 두 가지 상태 업데이트 모델 → 불변 갱신 / 변경 가능한 갱신
- 두 모델을 같이 사용하는 경우 혼동하지 않도록 주의
- 멘탈 모델 전환을 쉽게 할 수 있도록 Valtio의 상태와 리액트의 상태를 명확하게 분리

### 프락시 기반 렌더링 최적화의 장점

#### 네이티브 자바스크립트 함수를 사용 가능

- 배열에서 요소 제거 시

```tsx
// 변경 가능 갱신
array.splice(index, 1)

// 불변 갱신
[...array.slice(0, index), ...array.slice(index + 1)]
```

- 중첩된 객체의 값 변경

```tsx
// 변경 가능 갱신
state.a.b.c.text = 'hello'

// 불변 갱신
{
	...state,m
	a: {
		...state.a,
		b: {
			...state.a.b,
			c: {
				...state.a.b.c,
				text: 'hello'
			},
		},
	},
}
```

#### 코드 가독성 개선

- 프락시 기반의 Valtio

```tsx
const Component = () => {
  const { count } = useSnapshot(state)
  return <>{count}</>
}
```

- 선택자 기반의 Zustand

```tsx
const Component = () => {
  const count = useStore(state => state.count)
  return <>{count}</>
}
```

### 프락시 기반 렌더링 최적화의 단점

- 예측 가능성이 떨어짐
- 렌더링 최적화를 내부적으로 처리하기 떄문에 디버깅이 다소 어려움
