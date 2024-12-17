## Valtio 사용법?

Valtio는 Proxy를 활용해 상태 변경을 감지하는 라이브러리이다.

### 변경 감지 및 불변 상태 생성하기

```tsx
import { proxy, snapshot } from "valtio";

const state = proxy({ count: 0 });
const snap1 = snapshot(state);
```

두 상태는 모두 `{ count: 0 }` 로 동일 하지만 proxy는 변경 가능한 객체이지만 snap은 `Object.freeze`로 변경 불가능한 객체이다.

**이를 어디에 사용하는가?**

```tsx
const state2 = proxy({
  obj1: { c: 0 },
  obj2: { c: 0 },
});

const snap21 = snapshot(state2);

++state2.obj1.c;

const snap22 = snapshot(state2);
```

snap21과 snap22는 서로 다른 참조이기 때문에 `snap21 !== snap22`가 된다. 그렇지만 obj2는 변경하지 않았기 때문에 `snap21.obj2 === snap22.obj2` 가 된다.

즉, valtio는 스냅샷을 자동으로 생성하지만 필요한 경우에만 스냅샷을 만들어서 최적화를 진행한다.

**✋ fetch에 Proxy로 인터셉터 구현해보기**

[https://inpa.tistory.com/entry/JS-📚-자바스크립트-Proxy-Reflect-고급-기법](https://inpa.tistory.com/entry/JS-%F0%9F%93%9A-%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-Proxy-Reflect-%EA%B3%A0%EA%B8%89-%EA%B8%B0%EB%B2%95)

---

```tsx
// 요청 인터셉터 타입
type RequestInterceptor = (
  input: RequestInfo,
  init?: RequestInit
) => [RequestInfo, RequestInit?] | Promise<[RequestInfo, RequestInit?]>;
// 응답 인터셉터 타입
type ResponseInterceptor = (response: Response) => Response | Promise<Response>;
// 에러 인터셉터 타입
type ErrorInterceptor = (error: unknown) => void | Promise<void>;

class FetchWithProxy {
  /** 요청 인터셉터 */
  private requestInterceptors: Array<RequestInterceptor> = [];
  /** 응답 인터셉터 */
  private responseInterceptors: Array<ResponseInterceptor> = [];
  /** 에러 인터셉터 */
  private errorInterceptors: Array<ErrorInterceptor> = [];

  public interceptor = {
    // 요청 인터셉터 추가
    request: {
      use: (interceptor: RequestInterceptor) => {
        this.requestInterceptors.push(interceptor);
      },
    },
    // 응답 인터셉터 추가
    response: {
      use: (interceptor: ResponseInterceptor) => {
        this.responseInterceptors.push(interceptor);
      },
    },
    // 에러 인터셉터 추가
    error: {
      use: (interceptor: ErrorInterceptor) => {
        this.errorInterceptors.push(interceptor);
      },
    },
  };

  // Proxy로 fetch 감싸기
  public create() {
    // 인터셉터 처리하는 과정
    const fetchHandler = async (
      input: RequestInfo,
      init?: RequestInit
    ): Promise<Response> => {
      try {
        // 요청 인터셉터 실행
        for (const interceptor of this.requestInterceptors) {
          const result = await interceptor(input, init);
          input = result[0];
          init = result[1];
        }

        // fetch 호출
        let response = await fetch(input, init);

        // 응답 인터셉터 실행
        for (const interceptor of this.responseInterceptors) {
          response = await interceptor(response);
        }

        return response;
      } catch (error) {
        // 에러 인터셉터 실행
        for (const interceptor of this.errorInterceptors) {
          await interceptor(error);
        }
        throw error; // 에러를 호출자에게 다시 전달
      }
    };

    // Proxy로 fetch를 감싼 객체 반환
    return new Proxy(fetch, {
      apply: (_, __, args: [RequestInfo, RequestInit?]) =>
        fetchHandler(...args),
    });
  }
}

// FetchWithProxy 인스턴스 생성
const customProxyFetch = new FetchWithProxy();

// 예: Authorization 헤더 추가
function setAuthorizationHeader(input: RequestInfo, init?: RequestInit) {
  // 원하는 헤더 설정하기
  const updatedInit: RequestInit = {
    ...init,
    headers: {
      ...(init?.headers || {}),
      // Accept: "application/json",
      // Authorization: "인증 토큰",
    },
  };

  console.log("요청 인터셉터 실행", updatedInit);

  return [input, updatedInit] as [RequestInfo, RequestInit?];
}

// 예: JSON 자동 파싱
async function parseJSONData(response: Response) {
  if (response.ok) {
    console.log("응답 인터셉터 실행", "json 파싱하기");

    const data = await response.json();
    return data;
  }

  throw new Error(`HTTP error: ${response.status}`);
}

// 예: 401 에러 처리
function setAuthroizedError(error: unknown) {
  if (error instanceof Error && error.message.includes("401")) {
    console.warn("인증 오류");
  }
}

// 요청 인터셉터 등록
customProxyFetch.interceptor.request.use(setAuthorizationHeader);
// 응답 인터셉터 등록
customProxyFetch.interceptor.response.use(parseJSONData);
// 에러 인터셉터 등록
customProxyFetch.interceptor.error.use(setAuthroizedError);

// Proxy로 생성된 fetch 사용
const customFetch = customProxyFetch.create();

// 사용 예시
const data = await customFetch("https://jsonplaceholder.typicode.com/posts/1");
```

## Proxy 사용하여 최적화해보기

계속해서 만든 카운터 예제를 valtio를 사용해 만들어보자.

```tsx
const state = proxy({
  count1: 0,
  count2: 0,
});

const Counter1 = () => {
  // 변수명을 snap으로 하는게 관습
  // 이렇게 되면 스냅샷을 했기 때문에 동결된다.
  const snap = useSnapshot(state);
  const inc = () => ++state.count1;

  return (
    <>
      {snap.count1} <button onClick={inc}>+1</button>
    </>
  );
};
```

snap은 읽기만 하는 것이고, state에 접근하여 추적 정보를 감지하고 필요한 경우에 리렌더링을 하게 되는 것이다.

❓: 이전의 전역 상태라이브러리와 사용법은 매우 흡사하지만 동작방식이 독특하다고 느꼈다. 스냅샷을 떠서 변경된 값을 추적하여 리렌더링을 시키는 방식으로 이뤄진다.

## 투두로 만들기

여기도 마찬가지로 Todo를 기반으로 예시를 만드는데, 독특한 부분만 가져왔다.

```tsx
const createTodo = (title: string) => {
  state.todos.push({
    id: nanoid(),
    title,
    done: false,
  });
};

const removeTodo = (id: string) => {
  const index = state.todos.findIndex((item) => item.id === id);
  state.todos.splice(index, 1);
};

const toggleTodo = (id: string) => {
  const index = state.todos.findIndex((item) => item.id === id);
  state.todos[index].done = !state.todos[index].done;
};
```

todo를 만들고, 제거하고 완료처리하는 함수가 리엑트 외부에 존재하고 **불변성을 유지하면서 하는것이 아닌 그냥 `push`, `splice`를 사용해서 직접적으로 변경**한다.

### 리렌더링 개선하기

TodoItem만 리렌더링 되도록 개선해보자.

```tsx
const TodoItem = ({ id }: { id: string }) => {
  // 받아온 id를 통해 todo를 뽑아내고,
  const todoState = state.todos.find((todo) => todo.id === id);
  if (!todoState) {
    throw new Error("invalid todo id");
  }
  // 뽑아낸 투두를 스냅샷을 떠서 그려준다.
  const { title, done } = useSnapshot(todoState);
  return (
    <div>
      <input type="checkbox" checked={done} onChange={() => toggleTodo(id)} />
      <span
        style={{
          textDecoration: done ? "line-through" : "none",
        }}
      >
        {title}
      </span>
      <button onClick={() => removeTodo(id)}>Delete</button>
    </div>
  );
};

const TodoList = () => {
  const { todos } = useSnapshot(state);
  // id만 뽑아서 배열로 만들어준다.
  const todoIds = todos.map((todo) => todo.id);
  return (
    <div>
      {todoIds.map((todoId) => (
        <MemoedTodoItem key={todoId} id={todoId} />
      ))}
    </div>
  );
};
```

## 멘탈 모델

Valito에서 중요하게 생각해야 할 점은 “멘탈 모델” 을 생각해야 하는 것이다.

**Valtio에는 두 가지 상태 업데이트 모델이 있다.**

- **불변 갱신**
- **변경 가능한 갱신**

자바스크립트에서는 변경 가능한 갱신을 허용하지만 리액트에서는 불변 갱신을 중심으로 만들어짐.
따라서 두 멘탈 모델을 명확하게 분리해야 한다.

Valtio에서는 불변 갱신이 아닌 변경 가능한 갱신을 통해 쉽고 가독성이 좋게 상태를 관리할 수 있게 제공해준다.
(immer와 비슷한 느낌?)

```tsx
state.a.b.c.text = "hello";
```
