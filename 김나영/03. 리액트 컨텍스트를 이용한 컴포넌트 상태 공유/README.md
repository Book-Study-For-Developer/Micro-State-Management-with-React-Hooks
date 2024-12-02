## 3. 리액트 컨텍스트를 이용한 컴포넌트 상태 공유

- 컨텍스트로 props를 대신해 컴포넌트간에 데이터를 전달하는 것
- 사용자 정의 훅 => useContext와 useState(혹은 useReducer) 이용으로 전역 상태 제공

#### 리액트 컨텍스트

- 공급자는 중첩될 수 없고, 소비자 컴포넌트는 컴포넌트 트리 중에서 가장 가까운 공급자(Provider)를 선택해 컨텍스트 값(value)을 가져온다.

#### useContext와 함께 useState 사용하기

- App.tsx

```jsx
const [count, setCount] = useState(0);
...
return (
    <CountStateContext.Provider value={{count, stCount}}>
        <Parent />
    </CountStateContext.Provider>
)
```

- Component1.tsx

```jsx
const { count, setCount } = useContext(CountStateContext);
return <div>{count}</div>;
```

#### 전역 상태를 위한 컨텍스트 만들기

- 컨텍스트 공급자가 새로운 value를 갖게 되면 모든 소비자는 리렌더링됨
  -> 완화할 수 있는 해결책 2가지

1. 작은 상태 조각 만들기

```jsx
const Count1Context = createContext({ ... })
const Count2Context = createContext({ ... })

// counter1에서는 다른 컨텍스트에 대해서는 알지 못함
const Counter1 = () => {
    const [count1, setCount1] = useContext(Count1Context);
    return (...)
}

const Counter2 = () => {
    const [count2, setCount2] = useContext(Count2Context);
    return (...)
}

const Parent = () => (
    <div>
        <Counter1 />
        <Counter2 />
    </div>
)
```

다음으로, Count1Context를 위한 Count1Provider을 따로 정의함 (value, setValue를 전달)

```jsx
const Count1Provider = ({ children }) => {
  const [count1, setCount1] = useState(0);
  return (
    <Count1Context.Provider value={[count1, setCount1]}>
      {children}
    </Count1Context.Provider>
  );
};
```

```jsx
const App = () => (
    <Count1Provider>
        <Count2Provider>
            <Parent>
        </Count2Provider>
    </Count1Provider>
)
```

- 리렌더링 문제는 발생하지 않지만, Counter 1에서는 count1이 변경될 때만 리렌더링됨
- 각 상태에 대한 Provider를 만들 필요가 있음

---

> 추가

**전역 상태 관리와 유사한 역할을 하는 싱글톤 패턴**

- 오늘 회사에서 컨벤션(?)이자 반강제로 급하게 웹소켓을 사용하게 되었는데, 전역상태 관리와 비슷한 점이 있는 것 같아 가져와보았다. 다만 상태관리만을 이용하는 것과는 장단점이 있기 때문에 ... 조금 더 고민이 필요할 것 같다.

- 싱글톤 패턴을 사용하여 인스턴스를 하나만 생성하고, 전역적으로 상태를 관리함
- 여러 컴포넌트에서 하나의 소켓 인스턴스를 사용할 수 있게 됨
- getInstance() 메서드를 통해 애플리케이션 전역에서 동일한 인스턴스를 가져올 수 있음.

```jsx
class WebSocketService {
    private static instance: WebSocketService;
    private readonly BASE_URL = 'ws://localhost:8080/ws';
    private readonly RECONNECT_INTERVAL = 5000;
    private socket: WebSocket | null = null;
    private connected: boolean = false;
    ...

    private constructor() {
        this.connect();
    }

    public static getInstance(): WebSocketService {
        if (!WebSocketService.instance) {
            WebSocketService.instance = new WebSocketService();
        }

        return WebSocketService.instance;
    }

    ...

    public isConnected(): boolean {
        return this.connected;
    }
    ...
}
```

- 다만 단점이 있었다. 전역 상태 관리 라이브러리처럼 상태 변경에 따른 자동 UI 업데이트 기능은 제공하지 않음.
- 이 부분을 해결하기 위해 custom hook을 만들어서 페이지단에서 가져와서 사용함

```jsx
export const useWebSocket = () => {
  const [connected, setConnected] = useState<boolean>(false);
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    const webSocketService = WebSocketService.getInstance();
    const socket = webSocketService.getSocket();

    const handleOpen = () => {
      setConnected(true); // + handleClose 함수
    };

    const handleMessage = (event: MessageEvent) => {
      setMessages((prevMessages) => [...prevMessages, event.data]);
    };

    socket?.addEventListener('open', handleOpen); // close, message도 동일함

    return () => {
      // cleanup 함수들
    };
  }, []);

  return { connected, messages };
};
```

- 사용시

```jsx
const WebSocketComponent = () => {
  const { connected, messages } = useWebSocket();

  return (
    <div>
      <p>WebSocket is {connected ? 'connected' : 'disconnected'}</p>
      <ul>
        {messages.map((message, index) => (
          <li key={index}>{message}</li>
        ))}
      </ul>
    </div>
  );
};
```
