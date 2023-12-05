# Context API + useReducer

# 0. 도입 이유

리액트 개발을 하게 되면 우리는 수 많은 상태와 마주하게 되고, 이러한 상태 값들은 개발을 굉장히 복잡하게 만든다.

이러한 문제를 해결하기 위해 `Redux` `Recoil` `Mobx` `Zustand` 등 다양한 상태 관리 라이브러리 등이 등장했다.

하지만 이러한 상태 관리 라이브러리는 서버 상태와 동기화 작업이 어렵다는 이유로 서버 상태와 동기화를 쉽게 해주는 `react-query` `SWR` `RTK-query` 같은 데이터 패칭 라이브러리가 등장하는 계기를 마련해줬다.

그러나 이러한 데이터 패칭 라이브러리를 사용해도 `useState` 같은 훅을 이용해서 `props` 를 내리는 행위를 아예 안 할 수는 없었다. 그리고 이러한 `props drilling` 은 가뜩이나 복잡한 개발을 더 복잡하게 만드는 원흉이 되었다. 특히 `state` 를 변경하는 `setState` 함수도 `props` 로 전달하게 되면 전달하는 값이 너무 많아져 코드를 굉장히 지저분하게 만드는데 일조했다.

이러한 문제를 해결하기 위해 최근에 진행중인 프로젝트에 `Context API` 를 도입했다([링크](https://www.notion.so/ContextAPI-3dd7901c57f440ed90a2acfe73c74f97?pvs=21)) 

덕분에 `props drilling` 이슈는 어느 정도 해결됐으나 사실 궁극적인 해결책이라고 볼 수는 없다. 그러면 지금부터 `Context API` 에 대해 알아보고 궁극적인 문제를 해결해보도록 하자.

---

# 1. Context API

우선 `React` 공식 문서에 나와있는 `useContext`의 정의부터 살펴보자

> `useContext` is a React Hook that lets you read and subscribe to [context](https://react.dev/learn/passing-data-deeply-with-context) from your component.
> 

해당 정의를 보면 `useContext` 에는 **상태관리**라는 단어를 찾아 볼 수 가 없다. 흔히들 `Context API` 즉 `useContext` 를 이용해서 **상태관리**를 하고 있다고 생각하지만 `useContext` 자체는 상태관리 훅이 아니다.

`useContext` 는 컴포넌트 단계마다 일일이 `props`를 전달하지 않고 컴포넌트 트리 전체에 데이터를 제공할 수 있는 기능을 제공할 뿐이다.

그렇다면 `Context API`로 상태관리를 한다는건 무슨 의미냐?

이는 `useState`와 `useReducer`를 이용해서 직접 상태와 상태관리 함수들을 `Context API`를 통해 전달해 상태를 관리하는 행위라고 말할 수 있다.

<aside>
💡 물론 이번 프로젝트에서는 단순히 view페이지에서만 Context API를 도입해서 직접 상태관리를 수행할 기회는 없었지만 다음 프로젝트에서는 props drilling을 최소화 하기 위해 ContextAPI를 사용할 것이고  이를 위해 해당 문서를 작성했다.

</aside>

`useState`와 `Context API` 를 이용해서 상태를 관리하는 것은 어렵지 않다. 정확히 말하면 매우 직관적이라 복잡하지는 않다.  하지만 `useReducer` 는 자주 사용해본 훅이 아니기 때문에 이번에 `useReducer` 에 대해 파악해보고 다음 프로젝트 때  필요하면 적재적소에 도입해보도록 하자.

### 참고 사항

`createContext` 훅을 이용할 때 `defaultValue` 를 넣어줘야 하는 상황이 있다. 이전까지는 항상 본인은 빈 객체를 넣었었다

```tsx
export const BrandListContext = createContext({});
```

이렇게 작성한 이유는 생성된 `Context`에 `value`값을 넣을 때 기본값을 넣어주기 때문에 크게 신경쓰지 않았다. 

또한 함수 같은 경우는 아예 다른 함수가 들어가게 되어 `defaultValue` 에 빈 함수를 넣어주는 것 에  반감이 있었는데, 여러 예시를 보고 나니 선언할 때 함수 기본 값에도 `defaultValue`를 넣어줘야 할 것 같다.

- **BAD (defaultValue값을 안 넣어주고 있음)**
    
    ```tsx
    export const BrandListContext = createContext({});
    .
    .
    .
    .
    <BrandListContext.Provider
      value={{
    	  brandListSearchPrams: brandListSearchPrams,
    	  setBrandListSearchParams: setBrandListSearchParams,
    	  brandList: brandList,
    	}}
    >
    {children}
    </BrandListContext.Provider>
    ```
    
- **GOOD**
    
    ```tsx
    export const BrandListContext = createContext({
      brandListSearchPrams: {},
      setBrandListSearchParams: {},
      brandList: [],
    });
    .
    .
    .
    .
    <BrandListContext.Provider
      value={{
    	  brandListSearchPrams: brandListSearchPrams,
    	  setBrandListSearchParams: setBrandListSearchParams,
    	  brandList: brandList,
      }}
    >
    {children}
    </BrandListContext.Provider>
    ```
    

---

# 2. useReducer

`useReducer` 는 `react`의 내장 `Hook`중 하나로 `reducer`를 사용해서 컴포넌트의 상태 변경을 관리할 때 사용한다. `useState`보다 복잡한 상태 변경을 할 때 주로 사용한다.

## 문법

```tsx
const [state, dispatch] = useReducer(reducer, initialArg, init?)
```

해당 코드는 컴포넌트 최상단에 호출에서 사용한다.

```tsx
import { useReducer } from 'react';

function reducer(state, action) {
  // ...
}

function MyComponent() {
	const [state, dispatch] = useReducer(reducer, { name: "luv" });
  // ...
```

### 파라미터

- `reducer`
    - 상태가 어떻게 업데이트할 지 작성해 놓은 `reducer` 함수 자리, 순수 함수여야 하며, 인자로 상태와 액션을 받으며 다음 상태 값을 반환해야 한다. 상태와 액션은 어떤 타입도 가능합니다.
- `initialArg`
    - 초기 값, 초기 상태를 계산하는 방법은 다음 `init` 인자에 따라서 달라집니다.
- `init?`
    - 초기 값을 반환하는 초기화 함수. 즉 두 번째 인자에 작성 해 놓은 초기 값을 해당 자리의 함수가 받으며 리턴 되는 값이 `initialArg` 값으로 변한다.

### 반환

`useReducer` 는 두 값을 가지는 배열을 리턴 한다.

- `const [state, dispatch] = useReducer(reducer, { age: 42 });`
    - `state`
        - 현재 상태, 첫 번째 렌더 중 `init(initialArg)` 또는 `initialArg` 값으로 설정된다.
    - `dispatch`
        - 상태를 다른 값으로 업데이트하고 재 렌더링을 발생 시키는 `dispatch` 함수

## dispatch 함수

`useReducer` 에 의해 반환 되는 `dispatch` 함수는 상태를 다른 값으로 업데이트하고 재 렌더를 발생 시킨다. `dispatch` 함수의 유일한 인자로 액션(주로 객체) 를 전달한다.

```tsx
const [state, dispatch] = useReducer(reducer, { name: "luv" });

function handleClick() {
  dispatch({ type: 'change_name' });
  // ...
```

보통 `dispatch` 함수를 함수로 감싸서 해당 함수를 실행시킨다.

### 파라미터

- `action`
    - 사용자에 수행되는 액션, 어떤 값도 가능하다. 그러나 일반적은 컨벤션에 의하면 해당 액션을 구분할 수 있는 `type` 프로퍼티와 전달할 값을 `payload` 프로퍼티로 전달한다.

## 불필요한 초기 상태 값 렌더링 방지

```tsx
function createInitialState(information) {
  // ...
}

function TestComponent({ username }) {
  const [state, dispatch] = useReducer(reducer, createInitialState(information));
  // ...
```

리액트 컴포넌트 단에서 초기 값을 한 번 저장하고 다음 렌더링부터는 초기 값을 무시한다.

`createInitialState(information)` 의 결과가 초기 렌더링에서 사용되지만, 모든 렌더에서 해당 함수를 호출하고 있다. 만약 초기 값 계산 비용이 많이 들면 매우 낭비다.

이러한 문제를 해결하기 위해 `useReducer` 의 세 번째 인자로 `Initializer` 함수 즉 여기서는 `createInitialState` 함수를 전달한다.

```tsx
function createInitialState(information) {
  // ...
}

function TestComponent({ username }) {
  const [state, dispatch] = useReducer(reducer, information, createInitialState);
  // ...
```

이러한 방법을 사용하면 초기 값을 계산하면서 `createInitialState` 함수의 인자로 `information` 이 사용된다.

해당 방법을 사용함으로써 함수를 직접 호출 하는 것이 아닌, `useReducer`에게 초기값 함수와 해당 인자를 전달하여 초기화 이후 초기 상태가 다시 생성되지 않는다. (즉 `state` 가 바뀌면 해당 컴포넌트가 재 렌더링 되면서 쓸데없는 `createInitialState` 가 실행되는데 이러한 방법으로 하면 재렌더링 돼도 함수가 다시 실행되지 않는다)

만약 `Initializer` 함수에 인자가 필요없다면 `null` 값을 두 번째 인자에 넣으면 된다.

---

# useState vs useReducer

어떤 방식을 이용해서 상태 관리를 할지는 성능적 문제보다 상태 관리의 복잡도에 초점을 두고 선택하면 된다. 만약 별다른 로직이 필요하지 않는 상태 관리면 고민하지 말고 `useState`를 사용해서 `setState`를 전달 해주도록 하자.

공식 문서에서 제공하고 있는 두 `hook` 의 차이점을 살펴보도록 하자

|  | useState | useReducer |
| --- | --- | --- |
| 코드 크기 | 일반적인 경우 useState가 적다 | 하지만 경우에 따라 state를 변경하는 로직이 많다면 reducer로 해당 로직을 옮겨 코드를 줄일 수 있다. |
| 가독성 | 간단 할 때는 쉽지만,  복잡한 상태 변경일 때는 어렵다 | reducer 내부에 상태 변경 로직이 있기 때문에 상태 변경 로직을 완전히 분리 할 수 있다. |
| 디버깅 | useState는 디버깅이 어렵다 | useReducer를 이용하면 콘솔 로그를 이용해서 디버깅 가능 |
| 테스트 | 불가 | reducer는 순수 함수이기 때문에 개별적 테스트 가능 |
| 선호도 | 개인선호 | 개인선호 |

### useReducer의 적절한 사용 시점

1. 객체 또는 배열 타입의 상태 관리
    -  `useState` 에서는 객체 또는 배열의 상태관리를 할 때 불변성을 지키기 위해 이전 값을 이용해 깊은 복사와 같은 방식으로 상태를 변경했다. 이러한 로직은 `setState` 함수 안에서 작성할 경우 굉장히 복잡해 질 수 있기 때문에 `useReducer` 의 `reducer` 를 이용해서 로직을 분리시켜 관리 할 수 있다.
2. 상태 변경 로직 분리
    -  위의 이유와 마찬가지로 상태변경을 해야하는 조건이 굉장히 많을경우 `setState` 함수내에서 `switch` 문이나 `if` 문을 반복적으로 사용했던 경험이 있을 것이다. 그러면 `setState` 내부 로직이 굉장히 복잡해지기 때문에 `useReducer` 의 `reducer` 를 이용해서 로직을 분리시켜 관리 할 수 있다.