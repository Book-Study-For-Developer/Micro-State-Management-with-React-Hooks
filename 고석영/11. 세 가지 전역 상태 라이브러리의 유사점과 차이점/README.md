### Zustand vs Redux

|                    | Redux                                                                                                                               | Zustand                                                                                              |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| 디렉터리 구조      | `features` 디렉터리 구조 [(공식 문서)](https://redux.js.org/style-guide/#structure-files-as-feature-folders-with-single-file-logic) | 라이브러리 차원에서 디렉터리 구조에 대한 의견 제시는 없음 → 개발자 재량                              |
| Immer 사용 여부    | Redux Toolkit에서 기본적으로 Immer 사용 [(공식 문서)](https://ko.redux.js.org/style-guide/#use-immer-for-writing-immutable-updates) | Immer 사용은 선택 사항                                                                               |
| 컨텍스트 사용 여부 | 상태 전파 시 컨텍스트 사용                                                                                                          | 모듈 임포트 사용                                                                                     |
| 데이터 흐름        | 단방향 데이터 흐름을 기반 (Flux)                                                                                                    | 단방향 데이터 흐름을 기반으로 작동하지만, 라이브러리 차원에서 데이터 흐름을 엄격하게 강제하지는 않음 |

> Because of this, **we recommend that most applications should structure files using a "feature folder" approach** (all files for a feature in the same folder). Within a given feature folder, **the Redux logic for that feature should be written as a single "slice" file**, preferably using the Redux Toolkit `createSlice` API. (This is also known as the ["ducks" pattern](https://github.com/erikras/ducks-modular-redux)). While older Redux codebases often used a "folder-by-type" approach with separate folders for "actions" and "reducers", keeping related logic together makes it easier to find and update that code.

> [!NOTE]
>
> #### Redux에서 제안하는 디렉터리 구조
>
> ```txt
> /src
>   ├── index.tsx                 # React 컴포넌트 트리 엔트리포인트
>   ├── /app                      # 전역 설정 및 레이아웃
>   │   ├── store.ts              # Redux 스토어 설정
>   │   ├── rootReducer.ts        # 루트 reducer (옵션)
>   │   └── App.tsx               # 루트 React 컴포넌트
>   ├── /common                   # 재사용 가능한 훅, 컴포넌트, 유틸리티 등
>   └── /features                 # 특정 기능별로 사용하는 리덕스 관련 파일 및 컴포넌트 폴더들
>       └── /todos                # 하나의 기능에 대한 폴더
>           ├── todosSlice.ts     # Redux reducer 로직과 관련된 액션들 (RTK의 createSlice 사용)
>           └── Todos.tsx         # todos 기능에 해당하는 React 컴포넌트
> ```

> [!NOTE]
>
> #### “ducks” 패턴 (Redux Reducer Bundle)
>
> ![image.png](https://github.com/erikras/ducks-modular-redux/blob/master/duck.jpg)
>
> - Redux Toolkit을 사용할 경우 "ducks 패턴"을 따르는 구조를 추천
> - ducks 패턴 규칙
>   - rueducer를 export default 하기 (MUST)
>   - action creators를 named export 하기 (MUST)
>   - action type은 `'모듈명/reducer명/ACTION_TYPE'`과 같이 슬래시로 구분해서 작성 (MUST)
>   - action type은 영어 대문자 + snake_case로 작성 (MAY)
>
> ```tsx
> // widgets.js
>
> // Actions
> const LOAD = 'my-app/widgets/LOAD'
> const CREATE = 'my-app/widgets/CREATE'
> const UPDATE = 'my-app/widgets/UPDATE'
> const REMOVE = 'my-app/widgets/REMOVE'
>
> // Reducer
> export default function reducer(state = {}, action = {}) {
>   switch (action.type) {
>     // do reducer stuff
>     default:
>       return state
>   }
> }
>
> // Action Creators
> export function loadWidgets() {
>   return { type: LOAD }
> }
>
> export function createWidget(widget) {
>   return { type: CREATE, widget }
> }
>
> export function updateWidget(widget) {
>   return { type: UPDATE, widget }
> }
>
> export function removeWidget(widget) {
>   return { type: REMOVE, widget }
> }
>
> // side effects, only as applicable
> // e.g. thunks, epics, etc
> export function getWidget() {
>   return dispatch => get('/widget').then(widget => dispatch(updateWidget(widget)))
> }
> ```

### Jotai vs Recoil

|                                      | Recoil                                              | Jotai                                                                           |
| ------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------------------- |
| key 문자열의 존재                    | 아톰 생성 시 반드시 고유한 key 문자열 필요 (직렬화) | 생략 가능                                                                       |
| 통합된 `atom` 함수                   | `atom`, `selector`로 분리                           | 아톰과 파생 아톰의 정의를 하나의 `atom` 함수로 처리                             |
| 공급자 제거 모드(provider-less mode) | 최상위 공급자 필요 (`RecoilRoot`)                   | 공급자를 생략해도 자동으로 감지하여 제공하며, 선택적으로 `Provider`도 사용 가능 |

> [!NOTE]
>
> #### 왜 Jotai에서 고유한 키를 제거했을까?
>
> - [How Jotai Was Born](https://blog.axlight.com/posts/how-jotai-was-born/)
>
>   > One thing I didn’t like about Recoil was that an atom required a string key. The API didn’t feel right. While it’s understandable that it’s for serialization, that actually convinced me that Recoil would never allow omitting string keys. So, I decided to develop my own library.
>
> - 책에서도 설명하지만, Recoil에서 아톰 생성 시 문자열 키를 필수로 작성해야 한다는 점을 Jotai 개발 이유로 꼽고 있음
> - 아마.. 편의성 + 고유값을 가져야 하는 키 값이 재선언되면서 발생할 수 있는 에러 가능성 → [Issue #733](https://github.com/facebookexperimental/Recoil/issues/733#issuecomment-729255961)을 고려한 게 아닌가 싶음

### Valtio vs MobX

|                    | Valtio                         | MobX                             |
| ------------------ | ------------------------------ | -------------------------------- |
| 갱신 방식          | 클래스 기반 상태 관리 지원     | 객체 기반 상태 관리 지원         |
| 렌더링 최적화 방식 | 예측 가능성이 높은 옵저버 방식 | 동시성 렌더링에 친화적인 훅 방식 |

### Zustand, Jotai, Valtio 비교

|                              | Zustand        | Jotai          | Valtio                |
| ---------------------------- | -------------- | -------------- | --------------------- |
| 상태가 어디에 위치하는가?    | 모듈 상태      | 컴포넌트 상태  | 모듈 상태             |
| 상태 갱신 스타일은 무엇인가? | 불변 상태 모델 | 불변 상태 모델 | 변경 가능한 상태 모델 |
