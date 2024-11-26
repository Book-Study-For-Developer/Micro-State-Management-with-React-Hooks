## 01. 리액트 훅을 이용한 마이크로 상태 관리

### 📚 요약 정리 

- **마이크로 상태 관리**는 좀 더 목적 지향적이며 특정한 코딩 패턴과 함께 사용한다. 
- 리액트에서의 상태는 사용자 인터페이스(UI)를 나타내는 모든 데이터를 칭한다. 
- 마이크로 상태관리를 충족하기 위해서는 상태 읽기 , 갱신 , 상태 기반 렌더링이 기본적으로 필요하다.
- 사용자 정의 훅으로 분리했을때는 컴포넌트를 수정할 필요 없이 로직을 추가할 수 있다는 장점이 있다.
- `Suspense`는 기본적으로 비동기 처리에 대한 걱정 없이 컴포넌트를 코딩할 수 있다. 
- `동시성 렌더링`은 렌더링 프로세스를 청크라는 단위로 분할해서 CPU가 장시간 차단되는것을 방지하는 방법이다. 
- `베일아웃 (bailout)`은 리액트 기술 용어로 리렌더링을 발생시키지 않는 것을 의미한다. 
- `지연 초기화 (lazy initialization)`는 useState가 호출되기 전까지 init 함수는 평가되지 않고 느리게 평가된다. 즉, 컴포넌트가 마운트될때 한번 호출된다. 

### 📝 예제 정리 

```jsx
const Component = () => {
  const [count , setCount] = useState(0);

  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>Set Count to {count + 1}</button>
    </div>
  )
};

```
빠르게 두번 클릭했을때 한번만 증가한다. ==> 이걸 Batching이라고 표현해도 되는건가?!?

**useReducer로 useState를 구현**
```jsx
const useState = (initialState) => {
  const [state , dispatch] = useReducer(
    (prev , action) => 
      typeof action === 'function' ? action(prev) : action, initialState
  );

  return [state , dispatch]
} 
```
애석하게도 useState 내부가 useReducer로 이루어져있다는걸 책을 통해 알게 되었다,,

**useState로 useReducer 구현**

```jsx
const useReducer = () => {
  const [state , setState] = useState(initialState);
  const dispatch = (action) => setState(prev => reducer(prev, action));

  return [state , dispatch];
}
```

### 💡 알아보기

#### 1️⃣ useState와 useReducer의 차이점과 활용법
옛날에 아래 글을 읽고 useReducer를 활용해본 경험이 있는데 복잡한 상태와 검증이 필요할때 useState 대신 useReducer를 활용해보면 좋을 것 같다. 추가적으로 공식문서에서도 useState와 useReducer를 비교한 글이 있다.

🔗 링크
[useState 지옥에서 벗어나기](https://velog.io/@eunbinn/a-cure-for-react-useState-hell)
[useState와 useReducer 비교하기](https://ko.react.dev/learn/extracting-state-logic-into-a-reducer#comparing-usestate-and-usereducer)


#### 2️⃣ Suspense란 무엇인가?

Suspense는 기본적으로 리액트 패러다임에 걸맞는 선언적인 컴포넌트이다. 만일 Suspense를 사용하지 않고 로딩을 처리한다면 어떻게 될까? 

구현에 약간의 차이점이 있겠지만 대부분의 사람들은 아래와 같이 로딩 상태를 직접 관리하게 된다. 

```jsx
const UserList = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        setIsLoading(true);
        // API 호출 예시
        const response = await fetch('https://api.example.com/users');
        if (!response.ok) {
          throw new Error('Failed to fetch users');
        }
        const data = await response.json();
        setUsers(data);
      } catch (err) {
        console.log(err);
      } finally {
        setIsLoading(false);
      }
    };

    fetchUsers();
  }, []);

  if (isLoading) {
    return <LoadingSpinner>Loading...</LoadingSpinner>;
  }

  return (
    {users.map((user) => (
      <UserItem key={user.id}>
        <UserName>{user.name}</UserName>
        <UserEmail>{user.email}</UserEmail>
      </UserItem>
    ))}
  );
};
```

위와 같은 상황에서 Suspense에게 비동기 처리를 위임하여 보다 선언적이게 코드 관리가 가능하게 된다. 

```jsx
const WrapperComponent = () => {
	return (
		<Suspense fallback={<div>Loading!!!!</div>}>
			<UserList />	
		</Suspense>
	)
}

// 기존코드에서 Loading 관련된 상태가 없어지고 컴포넌트의 역할과 책임이 명확해진 것 같다.
const UserList = () => {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        // API 호출 예시
        const response = await fetch('https://api.example.com/users');
        if (!response.ok) {
          throw new Error('Failed to fetch users');
        }
        const data = await response.json();
        setUsers(data);
      } catch (err) {
        console.log(err);
      }
    };

    fetchUsers();
  }, []);

  return (
    {users.map((user) => (
      <UserItem key={user.id}>
        <UserName>{user.name}</UserName>
        <UserEmail>{user.email}</UserEmail>
      </UserItem>
    ))}
  );
};
```

여기서 잠깐! 위와 같이 Suspense로 감싸기만  한다고해서 Suspense는 모든 로딩을 처리할 수 있을까? **결론부터 말하면 아니다!**

Suspense는 기본적으로 하위 컴포넌트에서 Promise를 Catch하고 해당 Promise를 기반으로 Pending 상태가 된다면 Fallback을 fulfilled 상태가 된다면 하위 컴포넌트를 보여주게 된다.

UserList 컴포넌트는 사실 아래와 같은 로직이 되어야 할 것 같다. 

```jsx
const WrapperComponent = () => {
	return (
		<Suspense fallback={<div>Loading!!!!</div>}>
			<UserList />	
		</Suspense>
	)
}

const promiseWrapper = (promise) => {
  let status = "pending";
  let result;

  const s = promise.then(
    (value:any) => {
      status = "success";
      result = value;
    },
    (error:any) => {
      status = "error";
      result = error;
    }
  );

  return () => {
    switch (status) {
	    // pending 상태일때 promise를 상위 suspense가 catch할 수 있도록 던진다.
      case "pending":
        throw s;
      case "success":
        return result;
      case "error":
        throw result;
      default:
        throw new Error("Unknown status");
    }
  };
};

// 기존코드에서 Loading 관련된 상태가 없어지고 컴포넌트의 역할과 책임이 명확해진 것 같다.
const UserList = () => {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        // Promise 상태의 응답 객체를 promiseWrapper에게 전달하여 상위 Suspense가 Catch할 수 있게 한다. 
        const response = fetch('https://api.example.com/users').then(res => res.json());
        const data = promiseWrapper(response);
        setUsers(data);
      } catch (err) {
        console.log(err);
      }
    };

    fetchUsers();
  }, []);

  return (
    {users.map((user) => (
      <UserItem key={user.id}>
        <UserName>{user.name}</UserName>
        <UserEmail>{user.email}</UserEmail>
      </UserItem>
    ))}
  );
};
```

사실 위와 같이 항상 promiseWrapper를 사용하여 suspense에게 던지는 행위는 번거롭기 때문에 react-query와 같은 라이브러리를 활용하면 훨씬 수월하게 해결할 수 있다. 

react-query에서 suspense가 catch하기 위해서는 useSuspenseQuery를 사용하는데 해당 로직도 동일하게 Promise를 던지는 행위를 하고 있나 한번 살펴봤다.

`useSuspenseQuery.ts`

```jsx
export function useSuspenseQuery<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
>(
  options: UseSuspenseQueryOptions<TQueryFnData, TError, TData, TQueryKey>,
  queryClient?: QueryClient,
): UseSuspenseQueryResult<TData, TError> {
  if (process.env.NODE_ENV !== 'production') {
    if ((options.queryFn as any) === skipToken) {
      console.error('skipToken is not allowed for useSuspenseQuery')
    }
  }

  return useBaseQuery(
    {
      ...options,
      enabled: true,
      suspense: true,
      throwOnError: defaultThrowOnError,
      placeholderData: undefined,
    },
    QueryObserver,
    queryClient,
  ) as UseSuspenseQueryResult<TData, TError>
}
```

useSuspenseQuery를 살펴보니 useBaseQuery에 suspense 옵션을 활성화하는 작업을 진행하고

> 참고 (https://github.com/TanStack/query/blob/main/packages/react-query/src/useSuspenseQuery.ts)
    

`useBaseQuery.ts`

```jsx
 // Handle suspense
  if (shouldSuspend(defaultedOptions, result)) {
    throw fetchOptimistic(defaultedOptions, observer, errorResetBoundary)
  }
  
```

useBaseQuery 로직내에 shouldSuspend 로직이 true가 되면 fetchOptimistic 로직을 통해 throw하게 된다. 

> 참고 (https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts)
    

`suspense.ts`

```jsx
export const fetchOptimistic = <
  TQueryFnData,
  TError,
  TData,
  TQueryData,
  TQueryKey extends QueryKey,
>(
  defaultedOptions: DefaultedQueryObserverOptions<
    TQueryFnData,
    TError,
    TData,
    TQueryData,
    TQueryKey
  >,
  observer: QueryObserver<TQueryFnData, TError, TData, TQueryData, TQueryKey>,
  errorResetBoundary: QueryErrorResetBoundaryValue,
) =>
  observer.fetchOptimistic(defaultedOptions).catch(() => {
    errorResetBoundary.clearReset()
  })
```

fetchOptimistic은 react-query에 핵심인 observer내에 fetchOptimistic을 정의하고 

> 참고 (https://github.com/TanStack/query/blob/main/packages/react-query/src/suspense.ts)    
    
    

`queryObserver.ts`

```jsx
 fetchOptimistic(
    options: QueryObserverOptions<
      TQueryFnData,
      TError,
      TData,
      TQueryData,
      TQueryKey
    >,
  ): Promise<QueryObserverResult<TData, TError>> {
    const defaultedOptions = this.#client.defaultQueryOptions(options)

    const query = this.#client
      .getQueryCache()
      .build(this.#client, defaultedOptions)

    return query.fetch().then(() => this.createResult(query, defaultedOptions))
  }
```

fetchOptimistic 내부에서는 query.fetch()를 활용하게 되고 

```jsx
protected fetch(
    fetchOptions: ObserverFetchOptions,
  ): Promise<QueryObserverResult<TData, TError>> {
    return this.#executeFetch({
      ...fetchOptions,
      cancelRefetch: fetchOptions.cancelRefetch ?? true,
    }).then(() => {
      this.updateResult()
      return this.#currentResult
    })
  }

  #executeFetch(
    fetchOptions?: Omit<ObserverFetchOptions, 'initialPromise'>,
  ): Promise<TQueryData | undefined> {
    // Make sure we reference the latest query as the current one might have been removed
    this.#updateQuery()

    // Fetch
    let promise: Promise<TQueryData | undefined> = this.#currentQuery.fetch(
      this.options as QueryOptions<TQueryFnData, TError, TQueryData, TQueryKey>,
      fetchOptions,
    )

    if (!fetchOptions?.throwOnError) {
      promise = promise.catch(noop)
    }

    return promise
  }
```

fetch내에서 promise를 생성하여 비로소 throw 되는 것 같다.

최종적으로 정리하면 아래와 같은 로직을 타게 되는 것 같다.

- useBaseQuery의 shouldSuspend 조건이 true
- fetchOptimistic 호출하고 결과 throw
- observer.fetchOptimistic 실행
- observer.#executeFetch 실행
- #currentQuery.fetch()로 실제 데이터 요청 Promise 생성
- 이 Promise가 React Suspense에 잡힘

> 참고 (https://github.com/TanStack/query/blob/main/packages/query-core/src/queryObserver.ts)
    
    


