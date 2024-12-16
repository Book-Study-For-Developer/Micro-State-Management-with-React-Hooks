## 시나리오 3: Valtio

Valtio는 Zustand나 Jotai와 다르게 변경 가능한 갱신 모델을 기반으로 하는 또 다른 전역 상태 라이브러리이다. 리액트와의 통합을 위해 Valtio는 프락시를 사용해 변경 불가능한 스냅숏을 가져온다.

Valtio의 자동 렌더링 최적화는 상태 사용 추적이라는 기법을 활용하여 상태의 어느 부분이 사용되었는지 감지하고, 사용된 부분이 변경될 경우에만 컴포넌트를 리렌더링하게 된다.

### Valtio 살펴보기

불변 갱신 규칙을 따를 필요가 없는 경우를 상상해보면 아래와 같은 코드가 동작해야된다.

```js
++moduleState.count;
```

이러한 코드는 리액트의 불변성을 위배하기 때문에 올바르게 동작하지 않지만 이를 가능하게 하려면 프락시를 활용해서 가능하게 할 수 있다.

```js
const proxyObject = new Proxy({
  count: 0,
  text: 'hello',
}, {
  set: (target , prop, value) => {
    console.log('start', prop);
    target[prop] = value;
    console.log('end', prop);
  }
})
```
new Proxy를 통해 proxyObject를 생성했고 아래와 같이 결과를 확인할 수 있다.

```js
++proxyObject.count
start count
end count
1
```



