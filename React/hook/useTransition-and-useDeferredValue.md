# useTransition / useDeferredValue

## 0. 학습 목적

---

최근 `suspense`와 `error boundary` 와 같은 컴포넌트를 공부하면서 `useTransition` 과 `useDeferredValue` 에 대한 개념이 자주 등장했다.

아무래도 성능 개선에 주 목적을 가지고 있는 훅이라 평소에 잘 써볼 기회는 없어서 해당 훅의 컨셉과 개념에 대해 제대로 알고 있지 않아서 이번 기회를 통해서 두 훅을 공부하여 다음 프로젝트에 적용해볼 수 있도록 하자.

## 1. useTransition 이란?

---

`useTransition` 은 UI를 차단하지 않고 (즉 UI의 변경없이) 상태를 업데이트 할 수 있는 리액트에서 제공하는 훅이다.

<aside>
💡 **const [isPending, startTransition] = useTransition();**

</aside>

여기서 중요한 부분은 _UI를 차단하지 않고_ 라는 텍스트가 중요하다. 이는 **React 18** 에서 추가된 **Concurrent rendering** 과 긴밀하게 사용 가능하기 때문이다.

<aside>
💡 여기서 우선적으로 결론부터 말하면 useTransition을 적절하게 사용하면 Suspense에서 사용한 fallback 컴포넌트를 상황에 맞게 사용하여 클라이언트는 무턱대고 fallback 컴포넌트를 보는것이 안인 필요에 따라 기존의 UI를 볼 수 있다.

</aside>

기본적으로 특정 상태 값의 변경이 오래 걸릴 경우 해당 상태가 전부 업데이트 된 이후에 **HTML** 이렌더링 이 일어난다 이러한 이슈로 그 시간만큼 렌더 트리가 Block 되기 때문에 클라이언트는 해당 시간 동안 아무런 동작을 할 수 없는 상태이기 때문에 UX에 굉장히 치명적이다.

그러나 `useTransition` 을 사용하면 컴포넌트 최상위 수준에서 호출되어 `setTransition` 을 통해 우선순위가 낮은 상태 업데이트들을 `transition` 이라고 표시하여 UI가 렌더링될 때 우선순위에 따라서 업데이트 할 수 있도록한다. 이러한 방법으로 렌더링이 오래 걸리는 컴포넌트의 Block을 피할 수 있다.

```tsx
import { useState, useTransition } from "react";

export default function Home() {
  const [text, setText] = useState("");
  const [value, setValue] = useState("");
  const [isPending, startTransition] = useTransition();

  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    startTransition(() => {
      setText(e.target.value);
    });
    setValue(e.target.value);
  };

  console.log({ text, isPending });
  console.log({ value });

  return <input type="text" onChange={onChange} />;
}
```

위의 코드를 보면 `startTransition` 에 인자로 콜백 함수 내에서 `text` 라는 상태 값을 변경해주는 `setText` 함수를 실행시키고 있다.

이러한 로직을 통해 두 개의 상태 `text, value` 값 중 `text` 상태는 우선순위가 뒤로 밀리게 되어 `value` 상태가 먼저 변경된 이후 `text` 상태 값이 변경되고 `value` 가 변하는 동안 `text` 상태가 변경되는 그 대기 중 상태를 `isPending` 값으로 확인할 수 있다.

즉 `transition` 으로 표시된 상태들은 다른 상태들이 호출되면 잠시 중단도니고 다른 상태들의 업데이트가 완료되면 다시 시작한다. 이러한 우선순위 배정을 통해 UX를 더 개선시킬 수 있다.

### setTransition 유의사항

<aside>
💡

1. 동기 함수여야 한다.
2. `transition`으로 표시된 setState는 다른 setState 업데이트시 중단된다.
   - 다른 상태 업데이트가 있을 경우 그것을 먼저 처리한다는 뜻
3. 텍스트 입력을 제어하는 데 사용할 수 없다.

예시 코드를 살펴보면 `tab` 이라는 상태 값이 있다.

</aside>

### 예시 코드를 통해 살펴보기

```tsx
const App = () => {
  const [tab, setTab] = useState("about");

  return (
    <>
      {/* 탭을 클릭하면 렌더링할 탭 컴포넌트가 설정된다 */}
      <TabButton isActive={tab === "about"} onClick={() => setTab("about")}>
        About
      </TabButton>
      <TabButton isActive={tab === "posts"} onClick={() => setTab("posts")}>
        Posts (slow)
      </TabButton>
      <TabButton isActive={tab === "contact"} onClick={() => setTab("contact")}>
        Contact
      </TabButton>
      <hr />
      {/* 현재 탭에 따라 탭 컴포넌트가 렌더링 된다 */}
      {tab === "about" && <AboutTab />}
      {tab === "posts" && <PostsTab />}
      {tab === "contact" && <ContactTab />}
    </>
  );
};

const TabButton = ({ children, isActive, onClick }) => {
  const [isPending, startTransition] = useTransition();

  // 현재 탭이 활성화 되면 isActive 상태가 된다.
  if (isActive) {
    return <b>{children}</b>;
  }
  // 대기 중인 transition이 있다면 isPending이 된다.
  if (isPending) {
    return <b className="pending">{children}</b>;
  }
  /**
   * props로 받은 onClick 함수를 startTransition으로 감싸주기 때문에
   * onClick 함수(setTab)은 transition으로 설정되어 렌더링시 우선순위에서 밀리게 된다.
   * 그 결과 오랜시간이 걸리는 PostsTab 컴포넌트를 렌더링 하는 도중 다른 탭을 누르게 되면
   * PostsTab 컴포넌트의 렌더링을 멈추고 다른 컴포넌트를 렌더링하게 된다.
   **/
  const handleButtonClick = () => {
    startTransition(() => {
      onClick();
    });
  };
  return <button onClick={handleButtonClick}>{children}</button>;
};

const AboutTab = () => {
  return <p>Welcome to my profile!</p>;
};
const PostsTab = () => {
  const startTime = performance.now();
  while (performance.now() - startTime < 1) {
    // 1 ms 동안 아무것도 하지 않음으로써 매우 느린 코드를 실행한다.
  }
  return <p>PostsTab</p>;
};
const ContactTab = () => {
  return <p>ContactTab</p>;
};
const ContactTab = () => {
  return <p>ContactTab</p>;
};
```

그리고 `TabButton` 이라는 컴포넌트를 만들고 onClick `props`로 `setTab()` 함수를 전달하고 있다.

그리고 `tab` 상태 값에 따라서 각 `AboutTab PostTab ContactTab` 을 각각 렌더링 해주는 로직이다.

그런데 특이점은 `PostTab` 내부에서 느린코드를 실행해서

`PostTab` 을 렌더링 하는데는 1초의 시간이 걸린다.

렌더링 되고 있을 때 만약 다른 `TabButton` 을 클릭하면 `PostsTab` 이 렌더링이 완료 안되도 해당 컴포넌트의 렌더링을 멈추고 다른 컴포넌트를 렌더링한다.

## 2.Suspense간 관계

`useTransition`의 `startTransition`을 `Suspense`와 함께 사용한다면 불 필요한 `fallback` 컴포넌트의 렌더링을 막을 수 있다.

일반적으로 렌더링에 시간이 오래 걸리는 컴포넌트는 선언적으로 `Loading` 상태를 보여주기 위해 `Suspense`의 `fallback` 컴포넌트를 사용하게 된다.

하지만 만약에 `fallback` 컴포넌트가 아니라 기존의 화면을 그대로 보여주고 싶은 경우가 생길 수 있다.

가령 위의 코드에서는 단순히 탭 메뉴가 변경 되는데 탭 메뉴가 사라지고 `fallback` 컴포넌트가 나타나면 UX적으로 좋지 않을 수 있다.

이럴 때 오래 걸리는 상태 업데이트를 `startTransition`으로 감싸주게 되면 `Suspense` 내부적으로 `fallback` 컴포넌트가 아닌 이전 컨텐츠를 계속 표시해준다.

이는 사용자가 해당 컴포넌트가 모두 렌더링될 때 까지 기다릴 필요가 없으며 UX가 개선되는 효과가 있다.

## 3. \***\*useDeferredValue\*\***

---

**`useDeferredValue`** 상태를 바꾸는 함수의 우선순위를 밑으로 지정하는 것이 아닌 그 값 자체를 가장 낮은 우선순위로 지정하는 것이다. 즉

```tsx
import { useDeferredValue, useState } from "react";

export default function Home() {
  const [count1, setCount1] = useState(0);
  const [count2, setCount2] = useState(0);
  const [count3, setCount3] = useState(0);
  const [count4, setCount4] = useState(0);

  const deferredValue = useDeferredValue(count2);

  const onIncrease = () => {
    setCount1(count1 + 1);
    setCount2(count2 + 1);
    setCount3(count3 + 1);
    setCount4(count4 + 1);
  };

  console.log({ count1 });
  console.log({ count2 });
  console.log({ count3 });
  console.log({ count4 });
  console.log({ deferredValue });

  return <button onClick={onIncrease}>클릭</button>;
}
```

위와 같은 코드가 있다면 `count2` 값이 상태 변화 우선순위가 가장 낮아져 가장 나중에 변하게 된다.

## 4. 더 공부해야 할 부분

---

개념적인 부분은 다 이해가 되지만 가령 `setTab` 전달 받은 `onClick` 함수가 `startTransition` 의 콜백 함수에서 실행되면 그에 맞물린 `state` 값이 `isPending` 이 되어 그에 맞물린 `<PostsTab />` 컴포넌트가 렌더링 되기 전 에 변경되는게 맞는지 조금 헷갈린다….

사용에는 무리가 없어 보이나 이러한 부분을 더 공부해야 할 것 같다.
