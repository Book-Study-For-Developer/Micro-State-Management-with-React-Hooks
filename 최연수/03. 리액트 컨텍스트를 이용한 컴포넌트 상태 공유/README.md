# 03. 리액트 컨텍스트를 이용한 컴포넌트 상태 공유

> useState와 useContext, 전역 상태를 위한 컨텍스트, 컨텍스트 사용하기

<br/>

## 🔖 1. 정적 값을 이용해 useContext 사용하기

### Context와 Provider

- 컨텍스트를 사용하면 props를 사용하지 않고도 부모 컴포넌트에서 트리 아래에 있는 자식 컴포넌트로 값을 전달할 수 있다. 즉, props를 제거하고도 코드에 문맥을 부여할 수 있는 것이다.
- 글을 쓸 때도 문맥을 위해 주제와 그 주제가 하는 일을 명확히 해야 한다. 이처럼 컨텍스트에서 Provider(공급자)는 컨텍스트 값을 하위 컴포넌트들에게 제공하는 역할을 한다.
- 예를 들어, 아래 코드에서 컨텍스트는 이렇게 말한다. "자, 이 글(코드)에서 주인공(color)는 black(기본값)이었다가, 어떤 상황에서는 red, 또 어떤 상황에서는 green...." 이 때 color가 어떠한 색으로 될 수 있는지 그 값은 provider가 제공한다.

### 예시 코드

```jsx
import { createContext, useContext } from "react";

const ColorContext = createContext('black');

const Component = () => {
  const color = useContext(ColorContext);
  return <div style={{ color }}>Hello {color}</div>;
};

const App = () => (
  <>
    <Component />
    <ColorContext.Provider value="red">
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value="green">
      <Component />
    </ColorContext.Provider>
    <ColorContext.Provider value="blue">
      <Component />
      <ColorContext.Provider value="skyblue">
        <Component />
      </ColorContext.Provider>
    </ColorContext.Provider>
  </>
);

export default App;
```

<br/>

## 🔖 2. 추가적인 리렌더링에 관해

### 오버 엔지니어링? or not?

- 이 책에서는 중요하게 다룬 건 아니지만, 최근 고민했던 부분이었다.
- 리렌더링은 책에서 말하듯, 기술적으로 피해야 하는 불필요한 연산이다.
- 다만, 리팩토링을 하면서 드는 공수에 비해 성능에 크게 문제가 되지 않는 경우 어느정도 타협을 봤던 경험이 있다. 즉, 오버 엔지니어링을 굳이 하지 않은 것이다.
- 이력서 한 줄 채워넣을 수는 있겠으나, 한정된 시간 안에 더 중요한 것을 하는 것이 중요하다 생각했기 때문이다. 회사 차원에서도 충분히 이해해줄 것이다(?)

<br/>

## 🔖 3. 전역 상태를 위한 컨텍스트 만들기

### useReducer로 하나의 상태를 만들고 여러 개의 컨텍스트로 전파하기

- 하나의 전역 상태를 `useReducer`로 관리한다. 그리고 여러 개의 컨텍스트를 사용하여 상태의 일부를 필요한 컴포넌트들에게 배포한다.
- 상태 갱신 함수(`dispatch`)는 별도의 컨텍스트로 배포하여 필요할 때만 접근할 수 있게 한다.
- 핵심은, 불필요한 렌더링을 피하기 위해 여러 컨텍스트를 사용하는 것이다.

### 프로젝트에 적용해보기

```jsx
import React, { createContext } from 'react';

// 상태 조각에 대한 컨텍스트 생성
const FilesContext = createContext();
const CurrentDirectoryContext = createContext();
const LoadingContext = createContext();

// 상태 갱신 함수(dispatch)에 대한 컨텍스트 생성
const StorageDispatchContext = createContext();
```

- 예를 들어, 파일과 디렉토리를 관리하는 Storage 애플리케이션 (e.g. Google Drive)이 있다.
- 전역 상태로는 `files`: 현재 디렉토리에 있는 파일 목록, `curDir`: 현재 디렉토리 경로, `loading`: 파일 로딩 상태 등이 있을 것이다.
- 그리고 상태 갱신 함수로는 파일 및 디렉토리 관련 액션을 처리하기 위한 dispatch 함수가 있을 것이다.

<br/>

## 🔖 4. 

### 

<br/>

## 🔖 5. 

### 

<br/>

## 📚 References

- 