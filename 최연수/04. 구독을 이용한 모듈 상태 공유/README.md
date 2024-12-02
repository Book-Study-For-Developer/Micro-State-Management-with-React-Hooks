# 04. 구독을 이용한 모듈 상태 공유

> 전역 상태를 다루기 위한 모듈 상태 사용법, selector와 useSubscription 사용

<br/>

## 🔖 1. 모듈 상태

### 책 설명 풀어쓰기

- 모듈 상태는 함수 밖에서 정의된 변수들을 말한다. 즉, 파일의 최상위 레벨에서 선언된 변수들이다.
- 이런 변수들은 전역 변수이거나 해당 파일 내에서만 접근할 수 있는 변수일 수 있다.
- 리액트 컴포넌트에서 모듈 상태를 사용하려면 어떻게 해야 할까?
- 리액트 컴포넌트는 자체적인 상태와 생명주기를 가지고 있다. (그래서 이것을 관리하는 게 중요하고 그래서 지금 상태 관리를 공부하고 있지..)
- 하지만 컴포넌트 외부에 있는 모듈 상태를 사용하려면 그 상태의 변경을 감지하고 컴포넌트를 업데이트해야 한다.
- 이를 위해 구독(subscription)을 사용한다. 구독을 통해 모듈 상태가 변경될 때마다 컴포넌트가 그 변화를 알아채고 화면을 다시 렌더링할 수 있다.

<br/>

## 🔖 2. 리액트에서 전역 상태를 다루기 위한 모듈 상태 사용법

### 리액트에서 리렌더링을 일으키는 훅

```jsx
let count = 0;

const Component1 = () => {
  const [state, setState] = useStore(count);
  const inc = () => {
    count += 1;
    setState(count);
  }

  return (
    <div>{state} <button onClick={inc}>+1</button></div>
  );
};
```

- useState와 useReducer: 이 두 개가 모듈 상태에 반응한다.
- 즉, 버튼을 클릭하면 useState의 count가 반응(증가 또는 감소)하고, 컴포넌트가 리렌더링 된다.

<br/>

## 🔖 3. selector와 useSubscription 사용하기

### selector 사용 목적

- Selector를 사용하면 컴포넌트가 필요한 상태의 일부만 선택하여 사용할 수 있다.
- 이를 통해 불필요한 리렌더링을 방지하고 성능을 최적화할 수 있다.

### 프로젝트에 적용해 보기

```jsx
import React from 'react';
import useSubscription from './useSubscription';

const FileList = () => {
  const files = useSubscription((state) => state.files); // selector

  return (
    <ul>
      {files.map((file) => (
        <li key={file.id}>{file.name}</li>
      ))}
    </ul>
  );
};

export default FileList;
```

예를 들어, 위와 같이 컴포넌트에서 `useSubscription` 훅과 Selector를 사용하여 필요한 상태만 구독한다.
