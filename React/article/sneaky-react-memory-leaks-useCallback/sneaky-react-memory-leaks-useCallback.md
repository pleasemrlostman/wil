# 교활한 React 메모리 누수: useCallback과 클로저가 문제를 일으킬 수 있는 방법

> 💡 useCallback에 관련된 좋은 아티클을 발견해서 번역해봤다. 왜 uesCallback을 남발하면 안되는지 알려주는 좋은 예시를 확인할 수 있었다.

## **Sneaky React Memory Leaks: How `useCallback` and closures can bite you**

I work at [**Ramblr**](https://ramblr.ai/), an AI startup where we build complex React applications for video annotation. I recently encountered a complex memory leak that was caused by a combination of JavaScript closures and React's **`useCallback`** hook. Coming from a .NET background it took me quite some time to figure out what was going on, so I thought I'd share what I learned.

I added a brief refresher on closures, but feel free to skip that part if you're already familiar with how they work in JavaScript.

- *Follow-up article for React Query users:* [**Sneaky React Memory Leaks II: Closures Vs. React Query**](https://www.schiener.io/2024-05-29/react-query-leaks).
- *Interested how the React compiler will handle this?:* [**Sneaky React Memory Leaks: How the React compiler won't save you**](https://www.schiener.io/2024-07-07/react-closures-compiler).

---

## 교활한 React 메모리 누수: `useCallback`과 클로저가 문제를 일으킬 수 있는 방법

저는 Ramblr라는 AI 스타트업에서 근무하고 있으며, 비디오 주석을 위한 복잡한 React 애플리케이션을 개발하고 있습니다. 최근에 JavaScript의 클로저와 React의 `useCallback` 훅이 결합하여 복잡한 메모리 누수 문제를 경험하고 있습니다. .NET 배경을 가지고 있어서 이러한 현상을 이해하는 데 상당한 시간이 소요되고 있습니다. 그래서 제가 배운 내용을 공유하고자 합니다.

클로저에 대한 간단한 복습을 추가하고 있지만, JavaScript에서 클로저가 어떻게 작동하는지 이미 알고 계신다면 해당 부분은 생략하셔도 좋습니다.

- 후속 기사로는 React Query 사용자들을 위한 "Sneaky React Memory Leaks II: Closures Vs. React Query"를 다루고 있습니다.
- React 컴파일러가 이 문제를 어떻게 처리할지 궁금하신가요? "Sneaky React Memory Leaks: How the React compiler won't save you"라는 제목으로 다룰 예정입니다.

---

## A breif refresher on Closures

Closures are a fundamental concept in JavaScript. They allow functions to remember the variables that were in scope when the function was created. Here's a simple example:

```tsx
function createCounter() {
  const unused = 0; // This variable is not used in the inner function
  let count = 0; // This variable is used in the inner function
  return function () {
    count++;
    console.log(count);
  };
}

const counter = createCounter();
counter(); // 1
counter(); // 2
```

In this example, the **`createCounter`** function returns a new function that has access to the **`count`** variable. This is possible because the **`count`** variable is in the scope of the **`createCounter`** function when the inner function is created.

JavaScript closures are implemented using a context object that holds references to the variables in scope when the function was originally created. Which variables get saved to the context object is an implementation detail of the JavaScript engine and is subject to various optimizations. For example, in V8, the JavaScript engine used in Chrome, unused variables might not be saved to the context object.

Since closures can be nested inside other closures, the innermost closures will hold references (through a so-called scope chain) to any outer function scope that they need to access. For example:

```tsx
function first() {
  const firstVar = 1;
  function second() {
    // This is a closure over the firstVar variable
    const secondVar = 2;
    function third() {
      // This is a closure over the firstVar and secondVar variables
      console.log(firstVar, secondVar);
    }
    return third;
  }
  return second();
}

const fn = first(); // This will return the third function
fn(); // logs 1, 2
```

In this example, the **`third()`** function has access to the **`firstVar`** variable through the scope chain.

![https://www.schiener.io/assets/img/react-closures-scopes.png](https://www.schiener.io/assets/img/react-closures-scopes.png)

So, as long as the app holds a reference to the function, none of the variables in the closure scope can be garbage collected. Due to the scope chain, even the outer function scopes will remain in memory.

Check out this amazing article for a deep dive into the topic: [**Grokking V8 closures for fun (and profit?)**](https://mrale.ph/blog/2012/09/23/grokking-v8-closures-for-fun.html). Even though it's from 2012, it's still relevant and provides a great overview of how closures work in V8.

---

## **클로저에 대한 간략한 복습**

클로저는 JavaScript의 기본 개념입니다. 클로저는 함수가 생성될 때의 범위에 있는 변수들을 기억할 수 있게 해줍니다. 간단한 예를 들어보겠습니다:

```jsx
function createCounter() {
  const unused = 0; // 이 변수는 내부 함수에서 사용되지 않음
  let count = 0; // 이 변수는 내부 함수에서 사용됨
  return function () {
    count++;
    console.log(count);
  };
}

const counter = createCounter();
counter(); // 1
counter(); // 2
```

이 예제에서 `createCounter` 함수는 `count` 변수를 사용할 수 있는 새로운 함수를 반환합니다. 이는 `count` 변수가 내부 함수가 생성될 때 `createCounter` 함수의 범위에 있기 때문에 가능합니다.

JavaScript의 클로저는 함수가 원래 생성될 때 범위에 있는 변수에 대한 참조를 보유하는 컨텍스트 객체를 사용하여 구현됩니다. 어떤 변수가 컨텍스트 객체에 저장되는지는 JavaScript 엔진의 구현 세부 사항이며 다양한 최적화에 따라 달라질 수 있습니다. 예를 들어, Chrome에서 사용하는 V8 엔진에서는 사용되지 않는 변수가 컨텍스트 객체에 저장되지 않을 수 있습니다.

클로저는 다른 클로저 내부에 중첩될 수 있으므로, 가장 안쪽의 클로저는 필요한 외부 함수 범위에 대한 참조를 (소위 스코프 체인을 통해) 보유하게 됩니다. 예를 들어:

```jsx
function first() {
  const firstVar = 1;
  function second() {
    // 이는 firstVar 변수를 클로저로 캡처함
    const secondVar = 2;
    function third() {
      // 이는 firstVar와 secondVar 변수를 클로저로 캡처함
      console.log(firstVar, secondVar);
    }
    return third;
  }
  return second();
}

const fn = first(); // 이는 third 함수를 반환함
fn(); // 1, 2를 로그함
```

이 예제에서 `third()` 함수는 스코프 체인을 통해 `firstVar` 변수에 접근할 수 있습니다.

![https://www.schiener.io/assets/img/react-closures-scopes.png](https://www.schiener.io/assets/img/react-closures-scopes.png)

따라서 애플리케이션이 함수에 대해서 참조를 보유하고 있는 한, 클로저 범위 내의 어떤 변수도 가비지 컬렉션될 수 없습니다. 범위 체인 때문에, 외부 함수 범위조차도 메모리에서 유지됩니다.

이 주제에 대해 더 깊이 알고 싶다면, 훌륭한 기사를 확인해 보세요: "Grokking V8 closures for fun (and profit?)". 2012년에 작성되었지만 여전히 관련성이 있으며 V8에서 클로저가 어떻게 작동하는지에 대한 훌륭한 개요를 제공합니다.

---

## **Closures and React**

We heavily rely on closures in React for all functional components, hooks and event handlers. Whenever you create a new function that accesses a variable from the component's scope, for example, a state or prop, you are most likely creating a closure.

Here's an example:

```tsx
import { useState, useEffect } from "react";

function App({ id }) {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1); // This is a closure over the count variable
  };

  useEffect(() => {
    console.log(id); // This is a closure over the id prop
  }, [id]);

  return (
    <div>
      <p>{count}</p>
      <button onClick={handleClick}>Increment</button>
    </div>
  );
}
```

In most cases that in itself is not a problem. In the example above, the closures will be recreated on each render of **`App`** and the old ones will be garbage collected. That might mean some unnecessary allocations and deallocations, but those alone are generally very fast.

However, when our application grows and you start using memoization techniques like **`useMemo`** and **`useCallback`** to avoid unnecessary re-renders, there are some things to watch out for.

---

### 클로저와 React

우리는 React에서 모든 함수형 컴포넌트, 훅, 이벤트 핸들러에 대해 클로저를 많이 활용합니다. 컴포넌트의 범위에서 변수를 접근하는 새로운 함수를 생성할 때, 예를 들어 상태나 속성을 접근할 때, 대부분 클로저를 생성하는 것입니다.

아래는 예시입니다:

```jsx
import { useState, useEffect } from "react";

function App({ id }) {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1); // 이는 count 변수를 클로저로 캡처함
  };

  useEffect(() => {
    console.log(id); // 이는 id 속성을 클로저로 캡처함
  }, [id]);

  return (
    <div>
      <p>{count}</p>
      <button onClick={handleClick}>Increment</button>
    </div>
  );
}
```

대부분의 경우, 이는 문제가 되지 않습니다. 위의 예제에서 클로저는 `App`이 렌더링될 때마다 재생성되며, 이전 클로저는 가비지 컬렉션됩니다. 이는 불필요한 메모리 할당 및 해제를 의미할 수 있지만, 이러한 작업은 일반적으로 매우 빠릅니다.

하지만 애플리케이션이 커지면서 불필요한 리렌더링을 피하기 위해 `useMemo`와 `useCallback` 같은 메모이제이션 기법을 사용하기 시작하면 주의해야 할 몇 가지 사항이 있습니다.

---

## **Closures and `useCallback`**

With the memoization hooks, we trade better rendering performance for additional memory usage. **`useCallback`** will hold a reference to a function as long as the dependencies don't change. Let's look at an example:

```tsx
import React, { useState, useCallback } from "react";

function App() {
  const [count, setCount] = useState(0);

  const handleEvent = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  return (
    <div>
      <p>{count}</p>
      <ExpensiveChildComponent onMyEvent={handleEvent} />
    </div>
  );
}
```

In this example, we want to avoid re-renders of **`ExpensiveChildComponent`**. We can do this by trying to keep the **`handleEvent()`** function reference stable. We memoize **`handleEvent()`** with **`useCallback`** to only reassign a new value when the **`count`** state changes. We can then wrap **`ExpensiveChildComponent`** in **`React.memo()`** to avoid re-renders whenever the parent, **`App`**, renders. So far, so good.

But let's add a little twist to the example:

```tsx
import { useState, useCallback } from "react";

class BigObject {
  public readonly data = new Uint8Array(1024 * 1024 * 10); // 10MB of data
}

function App() {
  const [count, setCount] = useState(0);
  const bigData = new BigObject();

  const handleEvent = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  const handleClick = () => {
    console.log(bigData.data.length);
  };

  return (
    <div>
      <button onClick={handleClick} />
      <ExpensiveChildComponent2 onMyEvent={handleEvent} />
    </div>
  );
}
```

**Can you guess what happens?**

Since **`handleEvent()`** creates a closure over the **`count`** variable, it will hold a reference to the component's context object. And, even though *we never access **`bigData`** in the **`handleEvent()`** function*, **`handleEvent()`** will still hold a reference to **`bigData`** through the component's context object.

All closures share a common context object from the time they were created. Since **`handleClick()`** closes over **`bigData`**, **`bigData`** will be referenced by this context object. This means, **`bigData`** will never get garbage collected as long as **`handleEvent()`** is being referenced. This reference will hold until **`count`** changes and **`handleEvent()`** is recreated.

![https://www.schiener.io/assets/img/react-closures-bigObjectCapture.png](https://www.schiener.io/assets/img/react-closures-bigObjectCapture.png)

---

### 클로저와 useCallback

메모이제이션 훅을 사용하면 더 나은 렌더링 성능을 위해 추가적인 메모리 사용을 거래하게 됩니다. `useCallback`은 의존성이 변경되지 않는 한 함수에 대한 참조를 유지합니다. 예제를 살펴보겠습니다:

```jsx
import React, { useState, useCallback } from "react";

function App() {
  const [count, setCount] = useState(0);

  const handleEvent = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  return (
    <div>
      <p>{count}</p>
      <ExpensiveChildComponent onMyEvent={handleEvent} />
    </div>
  );
}
```

이 예제에서는 `ExpensiveChildComponent`의 리렌더링을 피하고자 합니다. 이를 위해 `handleEvent()` 함수의 참조를 안정적으로 유지하려고 합니다. `useCallback`을 사용하여 `handleEvent()`를 메모이제이션함으로써 `count` 상태가 변경될 때만 새로운 값을 재할당하도록 합니다. 그런 다음, `ExpensiveChildComponent`를 `React.memo()`로 감싸서 부모인 `App`이 렌더링될 때마다 리렌더링을 피할 수 있습니다. 지금까지는 잘 진행되고 있습니다.

하지만 예제에 약간의 변화를 추가해 보겠습니다:

```jsx
import { useState, useCallback } from 'react';

class BigObject {
  public readonly data = new Uint8Array(1024 * 1024 * 10); // 10MB의 데이터
}

function App() {
  const [count, setCount] = useState(0);
  const bigData = new BigObject();

  const handleEvent = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  const handleClick = () => {
    console.log(bigData.data.length);
  };

  return (
    <div>
      <button onClick={handleClick} />
      <ExpensiveChildComponent2 onMyEvent={handleEvent} />
    </div>
  );
}

```

**무슨 일이 벌어질지 짐작할 수 있나요?**

`handleEvent()`가 `count` 변수를 클로저로 캡처하기 때문에, 이 함수는 컴포넌트의 컨텍스트 객체에 대한 참조를 유지합니다. 그리고 `handleEvent()` 함수에서 `bigData`를 절대 사용하지 않더라도, 여전히 `handleEvent()`는 컴포넌트의 컨텍스트 객체를 통해 `bigData`에 대한 참조를 유지합니다.

모든 클로저는 생성된 시점의 공통 컨텍스트 객체를 공유합니다. `handleClick()`이 `bigData`를 클로저로 캡처하기 때문에, 이 컨텍스트 객체에서 `bigData`가 참조됩니다. 이는 `handleEvent()`가 참조되는 한 `bigData`가 가비지 컬렉션되지 않음을 의미합니다. 이 참조는 `count`가 변경되고 `handleEvent()`가 재생성될 때까지 유지됩니다.

![https://www.schiener.io/assets/img/react-closures-bigObjectCapture.png](https://www.schiener.io/assets/img/react-closures-bigObjectCapture.png)

---

## **An infinite memory leak with `useCallback` + closures + large objects**

Let us have a look at one last example that takes all the above to the extreme. This example is a reduced version of what I encountered in our application. So while the example might seem contrived, it shows the general problem quite well.

```tsx
import { useState, useCallback } from "react";

class BigObject {
  public readonly data = new Uint8Array(1024 * 1024 * 10);
}

export const App = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const bigData = new BigObject(); // 10MB of data

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // This only exists to demonstrate the problem
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(bigData.data.length);
  };

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```

In this example, we have two memoized event handlers **`handleClickA()`** and **`handleClickB()`**. We also have a function **`handleClickBoth()`** that calls both event handlers and logs the length of **`bigData`**.

**Can you guess what happens when we alternate between clicking the "Increment A" and "Increment B" buttons?**

Let's take a look at the memory profile in Chrome DevTools after clicking each of these buttons 5 times:

![https://www.schiener.io/assets/img/bigobject-leak-useCallback.png](https://www.schiener.io/assets/img/bigobject-leak-useCallback.png)

Seems like **`bigData`** never gets garbage collected. The memory usage keeps growing and growing with each click. In our case, the application holds references to 11 **`BigObject`** instances, each 10MB in size. One for the initial render and one for each click.

The retention tree gives us an indication of what's going on. Looks like we're creating a repeating chain of references. Let's go through it step by step.

**0. First render:**

When **`App`** is first rendered, it creates a *closure scope* that holds references to all the variables since we use all of them in at least one closure. This includes **`bigData`**, **`handleClickA()`**, and **`handleClickB()`**. We reference them in **`handleClickBoth()`**. Let's call the closure scope **`AppScope#0`**.

![https://www.schiener.io/assets/img/closure-chain-0.png](https://www.schiener.io/assets/img/closure-chain-0.png)

**1. Click on "Increment A":**

- The first click on "Increment A" will cause **`handleClickA()`** to be recreated since we change **`countA`** - let's call the new one **`handleClickA()#1`**.
- **`handleClickB()#0`** will *not* get recreated since **`countB`** didn't change.
- This means, however, that **`handleClickB()#0`** will still hold a reference to the previous **`AppScope#0`**.
- The new **`handleClickA()#1`** will hold a reference to **`AppScope#1`**, which holds a reference to **`handleClickB()#0`**.

![https://www.schiener.io/assets/img/closure-chain-1.png](https://www.schiener.io/assets/img/closure-chain-1.png)

**2. Click on "Increment B":**

- The first click on "Increment B" will cause **`handleClickB()`** to be recreated since we change **`countB`**, thus creating **`handleClickB()#1`**.
- React will *not* recreate **`handleClickA()`** since **`countA`** didn't change.
- **`handleClickB()#1`** will thus hold a reference to **`AppScope#2`**, which holds a reference to **`handleClickA()#1`**, which holds a reference to **`AppScope#1`**, which holds a reference to **`handleClickB()#0`**.

![https://www.schiener.io/assets/img/closure-chain-2.png](https://www.schiener.io/assets/img/closure-chain-2.png)

**3. Second click on "Increment A":**

This way, we can create an endless chain of closures that reference each other and never get garbage collected, all the while lugging around a separate 10MB **`bigData`** object because that gets recreated on each render.

![https://www.schiener.io/assets/img/closure-chain.png](https://www.schiener.io/assets/img/closure-chain.png)

---

### useCallback + 클로저 + 대용량 객체로 인한 무한 메모리 누수

이제 위의 모든 내용을 극단적으로 보여주는 마지막 예제를 살펴보겠습니다. 이 예제는 제가 우리 애플리케이션에서 경험한 사례의 축소판입니다. 비록 예제가 인위적으로 보일 수 있지만, 전반적인 문제를 잘 보여줍니다.

```jsx
import { useState, useCallback } from 'react';

class BigObject {
  public readonly data = new Uint8Array(1024 * 1024 * 10); // 10MB의 데이터
}

export const App = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const bigData = new BigObject(); // 10MB의 데이터

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // 문제를 보여주기 위해 존재하는 함수
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(bigData.data.length);
  };

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};

```

이 예제에서 우리는 두 개의 메모이제이션된 이벤트 핸들러 `handleClickA()`와 `handleClickB()`를 가지고 있습니다. 또한 두 이벤트 핸들러를 호출하고 `bigData`의 길이를 로그하는 `handleClickBoth()` 함수도 있습니다.

"Increment A" 버튼과 "Increment B" 버튼을 번갈아 클릭하면 무슨 일이 발생할지 짐작할 수 있나요?

각 버튼을 5번 클릭한 후 Chrome DevTools에서 메모리 프로필을 살펴보겠습니다:

![https://www.schiener.io/assets/img/bigobject-leak-useCallback.png](https://www.schiener.io/assets/img/bigobject-leak-useCallback.png)

### 대용량 객체 누수

`bigData`가 절대 가비지 컬렉션되지 않는 것처럼 보입니다. 각 클릭마다 메모리 사용량이 계속 증가합니다. 우리 경우, 애플리케이션은 각 10MB 크기의 11개의 `BigObject` 인스턴스에 대한 참조를 보유하게 됩니다. 초기 렌더링을 위한 하나와 클릭할 때마다 하나씩 더 생성됩니다.

보존 트리는 우리가 무슨 일이 벌어지고 있는지를 알려줍니다. 반복적인 참조 체인이 생성되고 있는 것처럼 보입니다. 단계별로 살펴보겠습니다.

1. **첫 렌더링**:

`App`이 처음 렌더링될 때, 모든 변수를 포함하는 클로저 범위를 생성합니다. 우리는 모든 변수를 최소한 하나의 클로저에서 사용하기 때문입니다. 여기에는 `bigData`, `handleClickA()`, `handleClickB()`가 포함됩니다. 우리는 이를 `handleClickBoth()`에서 참조합니다. 이를 `AppScope#0`이라고 부릅시다.

![https://www.schiener.io/assets/img/closure-chain-0.png](https://www.schiener.io/assets/img/closure-chain-0.png)

### 클로저 체인 0

1. **"Increment A" 클릭**:

"Increment A"를 첫 번째 클릭하면 `handleClickA()`가 재생성됩니다. 왜냐하면 `countA`가 변경되기 때문입니다. 새로운 것은 `handleClickA()#1`이라고 부릅시다.
`handleClickB()#0`는 `countB`가 변경되지 않기 때문에 재생성되지 않습니다.
그러므로 `handleClickB()#0`는 여전히 이전 `AppScope#0`을 참조하게 됩니다.
새로운 `handleClickA()#1`은 `AppScope#1`을 참조하며, 이 범위는 `handleClickB()#0`을 참조합니다.

### 클로저 체인 1

1. **"Increment B" 클릭**:

"Increment B"를 첫 번째 클릭하면 `handleClickB()`가 재생성됩니다. 왜냐하면 `countB`가 변경되기 때문입니다. 따라서 `handleClickB()#1`이 생성됩니다.
React는 `countA`가 변경되지 않았기 때문에 `handleClickA()`를 재생성하지 않습니다.
따라서 `handleClickB()#1`은 `AppScope#2`를 참조하며, 이 범위는 `handleClickA()#1`을 참조하고, 이는 다시 `AppScope#1`을 참조하며, 결국 `handleClickB()#0`을 참조하게 됩니다.

### 클로저 체인 2

1. **"Increment A"를 두 번째 클릭**:

이러한 방식으로 우리는 서로 참조하는 끝없는 클로저 체인을 생성할 수 있으며, 이들은 결코 가비지 컬렉션되지 않게 됩니다. 그 사이에 각각의 렌더링마다 재생성되는 별도의 10MB `bigData` 객체를 끌고 다니게 됩니다.

---

## **The general problem in a nutshell**

The general problem is that different **`useCallback`** hooks in a single component might reference each other and other expensive data through the closure scopes. The closures are then held in memory until the **`useCallback`** hooks are recreated. Having more than one **`useCallback`** hook in a component makes it super hard to reason about what's being held in memory and when it's being released. The more callbacks you have, the more likely it is that you'll encounter this issue.

---

### 일반적인 문제 요약

일반적인 문제는 하나의 컴포넌트 내에서 서로 다른 `useCallback` 훅이 서로 참조하거나 다른 대량의 데이터를 클로저 범위를 통해 참조할 수 있다는 것입니다. 이렇게 되면 클로저는 `useCallback` 훅이 재생성될 때까지 메모리에 유지됩니다. 컴포넌트에 `useCallback` 훅이 여러 개 있으면 메모리에 무엇이 저장되고 언제 해제되는지를 이해하기가 매우 어려워집니다. 콜백이 많을수록 이 문제에 직면할 가능성이 더 높아집니다.

---

## **Will this ever be a problem for you?**

Here are some factors that will make it more likely that you'll run into this problem:

1. You have some large components that are hardly ever recreated, for example, an app shell that you lifted a lot of state to.
2. You rely on **`useCallback`** to minimize re-renders.
3. You call other functions from your memoized functions.
4. You handle large objects like image data or big arrays.

If you don't need to handle any large objects, referencing a couple of additional strings or numbers might not be a problem. Most of these closure cross references will clear up after enough properties change. Just be aware that your app might hold on to more memory than you'd expect.

---

다음은 이 문제에 직면할 가능성을 높이는 몇 가지 요인입니다:

- 거의 재생성되지 않는 큰 컴포넌트를 가지고 있는 경우, 예를 들어, 많은 상태를 끌어올린 앱 쉘.
- 리렌더링을 최소화하기 위해 `useCallback`에 의존하는 경우.
- 메모이제이션된 함수에서 다른 함수를 호출하는 경우.
- 이미지 데이터나 큰 배열 같은 대량의 객체를 다루는 경우.

대량의 객체를 다룰 필요가 없다면, 몇 개의 추가 문자열이나 숫자를 참조하는 것은 문제가 되지 않을 수 있습니다. 대부분의 클로저 간의 교차 참조는 충분한 속성이 변경된 후에 사라집니다. 다만, 당신의 앱이 예상보다 더 많은 메모리를 사용하고 있을 수 있음을 인식하는 것이 중요합니다.

---

## **How to avoid memory leaks with closures and `useCallback`?**

Here are a few tips I can give you to avoid this problem:

_Tip 1: Keep your closure scopes as small as possible._

JavaScript makes it very hard to spot all the variables that are being captured. The best way to avoid holding on to too many variables is to reduce function size around the closure. This means:

1. _Write smaller components_. This will reduce the number of variables that are in scope when you create a new closure.
2. _Write custom hooks_. Because then any callback can only close over the scope of the hook function. This will often only mean the function arguments.

_Tip 2: Avoid capturing other closures, especially memoized ones._

Even though this seems obvious, React makes it easy to fall into this trap. If you write smaller functions that call each other, once you add in the first **`useCallback`** there is a chain reaction of all called functions within the component scope to be memoized.

_Tip 3: Avoid memoization when it's not necessary._

**`useCallback`** and **`useMemo`** are great tools to avoid unnecessary re-renders, but they come with a cost. Only use them when you notice performance issues due to renders.

_Tip 4 (escape hatch): Use **`useRef`** for large objects._

This might mean, that you need to handle the object's lifecycle yourself and clean it up properly. Not optimal, but it's better than leaking memory.

---

### 클로저와 useCallback으로 인한 메모리 누수를 피하는 방법

이 문제를 피하기 위한 몇 가지 팁을 드리겠습니다:

**팁 1: 클로저 범위를 가능한 작게 유지하세요.**

JavaScript는 캡처된 모든 변수를 파악하기 매우 어렵습니다. 너무 많은 변수를 유지하지 않으려면 클로저 주위의 함수 크기를 줄이는 것이 최선입니다. 즉:

- 더 작은 컴포넌트를 작성하세요. 이렇게 하면 새로운 클로저를 생성할 때 스코프에 있는 변수의 수가 줄어듭니다.
- 커스텀 훅을 작성하세요. 그러면 모든 콜백은 훅 함수의 스코프만 캡처하게 됩니다. 이 경우, 주로 함수 인수만 포함됩니다.

**팁 2: 다른 클로저, 특히 메모이제이션된 클로저를 캡처하지 마세요.**

이것은 분명해 보이지만, React에서는 쉽게 함정에 빠질 수 있습니다. 서로 호출하는 작은 함수를 작성하면, 첫 번째 `useCallback`을 추가했을 때, 컴포넌트 스코프 내의 모든 호출된 함수가 메모이제이션되도록 연결 반응이 발생합니다.

**팁 3: 필요하지 않을 때는 메모이제이션을 피하세요.**

`useCallback`과 `useMemo`는 불필요한 리렌더링을 피하기 위한 훌륭한 도구이지만, 비용이 따릅니다. 렌더링으로 인해 성능 문제가 발생한다고 느낄 때만 사용하세요.

**팁 4 (탈출구): 대용량 객체에 대해 useRef를 사용하세요.**

이 경우, 객체의 생명 주기를 직접 관리하고 적절히 정리해야 할 수도 있습니다. 최적의 방법은 아니지만, 메모리 누수를 피하는 것보다는 낫습니다.

---

## **Conclusion**

Closures are a heavily used pattern in React. They allow our functions to remember the props and states that were in scope when the component was last rendered. This can lead to unexpected memory leaks when combined with memoization techniques like **`useCallback`**, especially when working with large objects. To avoid these memory leaks, keep your closure scopes as small as possible, avoid memoization when it's not necessary, and possibly fall back to **`useRef`** for large objects.

Big thanks to David Glasser for his 2013 article [**A surprising JavaScript memory leak found at Meteor**](http://point.davidglasser.net/2013/06/27/surprising-javascript-memory-leak.html) that pointed me in the right direction.

---

### 결론

클로저는 React에서 자주 사용되는 패턴입니다. 클로저는 함수가 마지막으로 컴포넌트가 렌더링될 때의 props와 state를 기억할 수 있게 해줍니다. 그러나 이것이 `useCallback`과 같은 메모이제이션 기법과 결합될 때, 특히 대용량 객체를 다룰 때 예기치 않은 메모리 누수를 초래할 수 있습니다. 이러한 메모리 누수를 피하려면 클로저 범위를 가능한 작게 유지하고, 필요하지 않을 때는 메모이제이션을 피하며, 대용량 객체에는 `useRef`를 사용하는 것을 고려하세요.

2013년 David Glasser의 "A surprising JavaScript memory leak found at Meteor"라는 기사에 깊은 감사를 드립니다. 이 기사가 저를 올바른 방향으로 이끌어 주었습니다.
