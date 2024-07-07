# Next.js에서 tanstack(react)-query 어떻게 사용함?

# 0. 작성배경

이번에 개인적으로 진행하는 프로젝트는 next.js로 만들고 있다.

그런데 해당 프로젝트를 진행하면서 react-query 같은 라이브러리를 서버 컴포넌트에서 사용할 수 없는 문제를 직면했다.

그래서 당시에 라이브러리가 필요한 컴포넌트는 전부 `use clinet` 를 사용했는데 만들면서 느낀 부분은 이렇게 사용하면 next.js를 사용하는 이유가 뭐지?라는 의문이 생겼다.

하지만 역시 방법은 다 존재했다. 지금부터 Next.js에서 어떻게 `tanstack query`를 사용하는지 알바보고 실제 프로젝트에 적용해보자

# 1. 왜 필요할까?

그런데 굳이 `use clinet` 를 사용하면 쉽게 사용할 수 있는데 왜 서버에서 **tanstack query**를 사용하려고 할까.

## SEO 최적화

서버에서 tanstack query를 사용하면 SSR이나 SSG 할 때 해당 페이지에 필요한 데이터를 미리 받아올 수 있다. 그러면 클라이언트에게 서버에서 받은 데이터를 메타태그에 넣어 HTML을 만들 수 있으니까 SEO에 더 유리하다.

## 초기 로딩시간 단축 / 초기 데이터 로드 상태 관리

클라이언트 컴포넌트에서 데이터를 가져오게 되면 로딩상태 및 오류처리를 클라이언트에서 관리해야하는데 서버 컴포넌트로 미리 데이터를 가져오면 클라이언트에서는 이미 데이터가 박혀있는 HTML으로 페이지를 렌더링 할 수 있어서 UX를 향상 시킬 수 있다.

# 2. 어떻게 할까?

## 1. initialData를 사용하기

### 사용법

next.js의 `getServerSideProps` 와 `useQuery` 의 initialData를 이용해서 해당 로직을 처리할 수 있다. 가령 아래의 코드를 살펴보자 \*\*\*\*

```dart
export async function getServerSideProps() {
  const samples = await getSamples()
  return { props: { samples } }
}

function Posts(props) {
  const { data } = useQuery({
    queryKey: ['samples'],
    queryFn: getSamples,
    initialData: props.sample,
  })
}
```

아래의 방법을 이용하면 서버에서 미리 `getSamples` 함수가 실행되고 해당 함수에서 받은 데이터를 프롭스로 전달해준다 이후 클라이언트가 렌더링될 때 프롭스로 전달받은 데이터를 `initialData` 에 추가해줘 서버에서 받은 데이터를 `useQuery`가 사용할 수 있게된다.

### 문제점

하지만 해당 방법은 간편하지만 여러가지 문제점이 있다.

- 만약 `useQuery` 훅을 사용하는 컴포넌트가 트리 깊숙하게 즉 depth가 매우 깊다면 해당 서버에서 받은 데이터를 심연의 depth까지 전달해주는데 문제가 생긴다.
- 다양한 위치에서 같은 쿼리로 `useQuery` \*\*\*\*를 호출하는 경우 `initialData` 를 한개의 `useQuery` 에만 전달하는건 App이 변경될 때 문제가 생긴다.

  - 해당 부분이 조금 어렵게 느껴질 수 있는 쉽게 설명하면 `useQuery`는 동일한 `queryKey` 가 있다면 캐싱된 데이터를 이용해서 동일한 데이터를 사용할 수 있다. 하지만 `initialData` 를 한개의 `useQuery`에만 전달하는 경우 데이터 정합성이 틀릴 이슈가 발생할 수 있다. 그렇다고 모든 `useQuery` 에 `initialData` 를 전달하는 방법도 꽤나 번거로운 작업이다. 그러나 이렇게만 작성해도 글로만 보기 때문에 크게 와닿지 않는다.
  - **초기 데이터와 캐시 불일치 문제**
    - 설정된 `initialData` 가 캐시에 반영되지 않는 문제
      - `initialData` 는 컴포넌트가 처음 마운트 될때 사용된다.
      - 다른 위치에 있는 동일한 쿼리는 `initialData` 를 사용하지 않는다.
      - 즉 A 컴포넌트에서 같은 키값의 `useQuery`를 사용하고 `initialData` 를 설정햇는데 A 컴포넌트가 제거되거나 이동된다면 동일한 쿼리키를 사용하는 B 컴포넌트에서는 해당 `initialData` 값을 가지지 못한다.
  - **새로운 데이터가 캐시를 덮어쓰지 않음**

    - 서버에서 새로 데이터를 받아와도 즉 새롭게 신선한 데이터를 받아와도 기존 캐시된 데이터를 덮어쓰지 않는다.

      - 서버에서 새로운 데이터를 가져와도 클라이언트 캐시가 업데이트 되지 않는다.
      - `initialData` 는 최초 렌더링에만 사용되기 때문에 이후 데이터를 갱신하거나 유지하지 않는다.
      - 즉 새로 데이터를 가져와 `initialData` 로 설정했지만 기존 캐시가 새로운 데이터를 덮어 쓰지 않는다.

        ```dart
        import { useQuery } from '@tanstack/react-query';
        import { fetchMyData } from './api';

        function ComponentA() {
          const { data } = useQuery(['myData'], fetchMyData, {
            initialData: { id: 1, name: 'Initial Data' },
          });

          return <div>{data.name}</div>;
        }
        ```

        ```dart
        import { useQuery } from '@tanstack/react-query';
        import { fetchMyData } from './api';

        function ComponentB() {
          const { data } = useQuery(['myData'], fetchMyData);

          return <div>{data.name}</div>;
        }
        ```

        - 초기로드
          - 컴포넌트 A가 먼저 렌더링되고 `initialData` 가 사용된다 그리고 쿼리키는 **myData**이다
          - 컴포넌트 B는 같은 쿼리키를 사용하기 때문에 리액트 쿼리가 캐싱하고 있는 데이터를 사용한다
          - 만약 컴포넌트 A가 제거된다면 `initialData` 가 더이상 사용되지 않는다. B는 동일한 쿼리키를 사용하지만 `initialData` 가 존재하지 않기 때문에 새로운 데이터를 가져와야함

### 해결책

그렇다면 도대체 어떻게 이 문제를 해결 할 수 있을까? 다행히 tanstack query 에서는 [\*\*Hydration API](https://tanstack.com/query/v5/docs/framework/react/guides/ssr#using-the-hydration-apis)\*\* 라는 기능을 제공해준다.

## 2. Hydration API 사용하기

- **hydration api**는 **dehydrate**와 **hydrate**라는 함수를 제공하는데 해당 함수를 통해서 위의 과정(서버에서 데이터 받아서 useQuery가 사용할 수 있게 하는)을 간소화 하는데 각 api는 정확히 무엇이고 어떻게 사용하는지 보도록 하자

### **Dehydrate**

- 해당 api는 서버에서 **react-query**의 상태를 클라이언트로 전송할 수 있는 형태로 만들기 위해서 사용된다. 서버에서 데이터를 가져온 후, 해당 데이터를 직렬화하여 클라이언트 컴포넌트에게 전달한다. 직렬화된 데이터는 **dehydrateState** 형태로 표현되고 클라이언트 컴포넌트에서 **hydrate api**를 통해 다시 **react-query** 상태로 변환된다.

### **Hydrate**

- **hydrate**는 클라이언트 컴포넌트에서 직렬화된 상태값을 받아 이를 **react-query**의 상태로 변환한다. 이 과정에서 서버에서 미리 받아온 데이터를 클라이언트 쿼리 캐시에 적용하여, 클라이언트 컴포넌트가 네트워크 요청없이 해당 데이터를 사용할 수 있게 해준다.

### 사용법

그렇다면 해당 api를 어떻게 사용할 수 있을까? 다음의 코드를 살펴보도록 하자 (해당 과정은 우선 page router 방법으로 시연하도록 하겠다)

1. 프레임워크 로더 함수에서 **`const queryClient = new QueryClient(options)`**를 생성한다. 즉 `_app.tsx` 파일에서 쿼리클라이언트 인스턴스를 생성한다. 이때 여기서 중요한 부분은 인스턴스가 앱이 렌더링 될 때마다 재생성 되는걸 막기 위해 `useState` 안에서 생성해 주도록 하자.

   ```tsx
   import "@/styles/globals.css";
   import type { AppProps } from "next/app";
   import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
   import { useState } from "react";

   export default function App({ Component, pageProps }: AppProps) {
     const [queryClient] = useState(
       () =>
         new QueryClient({
           defaultOptions: {
             queries: {
               // SSR에서는 클라이언트에서 즉시 재요청하는 것을 피하기 위해,
               // default staleTime을 0보다 높게 설정하는 것이 일반적입니다.
               staleTime: 60 * 1000,
             },
           },
         })
     );

     return (
       <QueryClientProvider client={queryClient}>
         <Component {...pageProps} />
       </QueryClientProvider>
     );
   }
   ```

2. 앱 로더 함수에서 미리 가져오고 싶은 각 쿼리에 대해서 `await queryClient.prefetchQuery({queryKey:[...], queryFn: 함수})` 를 실행한다.

   1. 가능하면 `await Promise.all(...)`을 사용해 쿼리들을 병렬로 가져와라
   2. prefetch가 아닌 쿼리들이 있어도 괜찬하다. 해당 쿼리들은 서버에서 렌더링 되지않고 애플리케이션에 hydration 작업이 이루어진 이후 클라이언트에서 가져올 것이다. 이는 사용자 상호작용 후에만 표시되거나 페이지 하단에 있어 더 중요한 콘텐츠의 로딩을 방해하지 않기 위해서 사용된다.

   - 공식 홈페이지에 있는 문장을 해석했지만 2-b의 내용은 도무지 잘 이해가 되지 않을 수도 있는데 간단하게 말하면 ssr에서 미리 prefetch로 데이터를 받아와도 되고 클라이언트 사이드에서 데이터를 받아와도 크게 상관없다는 말이다. 그러므로 걍 필요한 순간에 적절하게 사용하라는 뜻이다.

     ```tsx
     // 서버 사이드에서 미리 가져오는 쿼리
     await queryClient.prefetchQuery("importantData", fetchImportantData);

     // 클라이언트 사이드에서 나중에 가져오는 쿼리
     const { data: lessImportantData } = useQuery(
       "lessImportantData",
       fetchLessImportantData
     );
     ```

   - 그렇다면 이제 직접 코드를 작성해 확인해보도록 하자.

     ```tsx
     // pages/posts.jsx

     import {
       dehydrate,
       HydrationBoundary,
       QueryClient,
       useQuery,
     } from "@tanstack/react-query";

     // 이것은 getServerSideProps에서도 동일할 수 있습니다.
     export async function getStaticProps() {
       const queryClient = new QueryClient();

       await Promise.all([
         queryClient.prefetchQuery({
           queryKey: ["posts"],
           queryFn: getPosts,
         }),
         queryClient.prefetchQuery({
           queryKey: ["posts-comments"],
           queryFn: getComments,
         }),
       ]);

       return {
         props: {
           dehydratedState: dehydrate(queryClient),
         },
       };
     }

     function Posts() {
       // useQuery는 <PostsRoute>의 더 깊은 자식에서도 마찬가지로 사용될 수 있으며,
       // 어느 방식이든 데이터는 즉시 사용 가능합니다.
       const { data: postsData } = useQuery({
         queryKey: ["posts"],
         queryFn: getPosts,
       });

       // 이 쿼리는 서버에서 미리 가져오지 않았으며, 클라이언트에서 시작될 때까지 요청하지 않을 것입니다.
       // 이 두 가지 패턴을 혼합해서 사용하는 것은 문제가 없습니다.
       const { data: commentsData } = useQuery({
         queryKey: ["posts-comments"],
         queryFn: getComments,
       });

       // ...
     }

     export default function PostsRoute({ dehydratedState }) {
       return (
         <HydrationBoundary state={dehydratedState}>
           <Posts />
         </HydrationBoundary>
       );
     }
     ```

     - 해당 코드를 통해서 `getPosts` 와`getComments` 를 함수를 통해 서버에서 데이터를 받아오고 받아온 데이터는 **dehydrate** 하여 **props**로 전달한다. 이후 클라이언트 컴포넌트에서는 쿼리키를 이용해서 해당 데이터에 접근할 수 있다.

   ### 보일러 플레이트 줄이기

   해당 코드는 문제 없이 잘 작동하지만 매번 **~~Route** 해당 코드에서는 `PostRoute` 를 작성해서 `HydrationBoundary` 로 감싸주는 코드가 보일러플레이트라고 느껴질 수 있다. 이러한 이슈를 해결하기 위해서 코드를 조금만 수정해주자

   ```tsx
   // _app.tsx

   import "@/styles/globals.css";
   import type { AppProps } from "next/app";
   import {
     QueryClient,
     QueryClientProvider,
     HydrationBoundary,
   } from "@tanstack/react-query";
   import { useState } from "react";

   export default function App({ Component, pageProps }: AppProps) {
     const [queryClient] = useState(
       () =>
         new QueryClient({
           defaultOptions: {
             queries: {
               // SSR에서는 클라이언트에서 즉시 재요청하는 것을 피하기 위해,
               // default staleTime을 0보다 높게 설정하는 것이 일반적입니다.
               staleTime: 60 * 1000,
             },
           },
         })
     );

     return (
       <QueryClientProvider client={queryClient}>
         <HydrationBoundary state={pageProps.dehydratedState}>
           <Component {...pageProps} />
         </HydrationBoundary>
       </QueryClientProvider>
     );
   }
   ```

   ```tsx
   // posts.tsx

   import { getComments, getPosts, getUsers } from "@/http";
   import { dehydrate, QueryClient, useQuery } from "@tanstack/react-query";

   // 이것은 getServerSideProps에서도 동일할 수 있습니다.
   export async function getServerSideProps() {
     const queryClient = new QueryClient();
     await queryClient.prefetchQuery({
       queryKey: ["posts"],
       queryFn: getPosts,
     });
     return {
       props: {
         dehydratedState: dehydrate(queryClient),
       },
     };
   }

   export default function Posts() {
     const { data: postsData } = useQuery({
       queryKey: ["posts"],
       queryFn: getPosts,
     });
     return (
       <>
         <div className="flex flex-col">
           {postsData?.map((value: any) => {
             return <p>{value.body}</p>;
           })}
         </div>
       </>
     );
   }
   ```

   거창하게 말했지만 사실 별거 없다 기존에 사용하던 `PostsRoute` 를 제거하고 Posts를 `export default` 해주고 `HydrationBoundary` 를 `_app.tsx` 즉 프레임워크 로더 함수에서 실행시키면 된다.

# 3. App 라우터에서는 어떻게 사용할까?

사실 지금까지 했던 구조는 모두 page 라우터를 기반으로 작성했다 하지만 현재 Next.js는 공식 홈페이지에서도 App 라우터 사용을 권장하고 있다.

또한 위의 코드들은 현재 로직이 단순하여 큰 문제가 없어 보이지만 나중에 사용해야 하는 queryKey와 서버와 통신할 api가 늘어날 수록 유지보수가 어렵다는 단점이 존재한다.

이러한 문제를 해결하기 위해 뛰어난 블로그 글을 찾았다.

블로그에 작성해주신 코드를 분석하면서 현재 진행중인 프로젝트에 적용해보도록 하자.

## 초기세팅

우선 `QueryClient` 를 만들어줄 클라이언트 컴포넌트를 하나 생성해준다.

```tsx
"use client";
import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

export default function ReactQueryProviders({
  children,
}: React.PropsWithChildren) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,
          },
        },
      })
  );
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

앞서 말한대로 매번 `QueryClient` 인스턴스를 렌더링할 때마다 만들면 안되므로 `useState`의 내부 콜백함수의 리턴값으로 인스턴스를 생성해준다. 그리고 `QueryClientProvider` 컴포넌트에 `client` 값으로 프롭스를 전달해주고 내부에 `children`을 받아준다.

이후 서버컴포넌트인 `layout.tsx` 에서 방금생성한 컴포넌트를 이용해서 초기 세팅을 마무리 해준다.

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import ReactQueryProviders from "@/app/_hooks/useReactQuery";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="ko">
      <body className={inter.className}>
        <ReactQueryProviders>{children}</ReactQueryProviders>
      </body>
    </html>
  );
}
```

그리고 이전과 다르게 로직을 도메인별로 불리시키기 위해 **\_service**라는 폴더를 생성해주었다.

해당 폴더에는 각 도메인에 필요한 서버와 통신 하는 함수, 쿼리 키, 쿼리 옵션 그리고 react-query의 useQuery를 사용 할 때 서비스별로 Colocate 시켜놓은 파일을 생성했다. 해당 파일은 클라이언트 컴포넌트에서 주로 사용할 예정인데 각 파일들을 보면서 살펴보도록하자

우선 서버와 통신하는 함수들을 생성해보자. 이전과는 다른 부분은 해당 함수들을 class의 메서드로 구현하여 조금도 구조적으로 관리할 수 있도록 했다.

```tsx
import axios, {
  AxiosInstance,
  AxiosResponse,
  InternalAxiosRequestConfig,
} from "axios";

class Service {
  protected http: AxiosInstance;

  constructor(baseURL: string) {
    this.http = axios.create({
      baseURL,
    });
    // 요청 인터셉터
    this.http.interceptors.request.use(
      (config: InternalAxiosRequestConfig) => {
        return config;
      },
      (error) => {
        return Promise.reject(error);
      }
    );
    // 응답 인터셉터
    this.http.interceptors.response.use(
      (response: AxiosResponse) => {
        return response;
      },
      (error) => {
        return Promise.reject(error);
      }
    );
  }
}

export default Service;
```

우선 서비스 클래스를 생성해주었다. 서비스 클래스에서는 http라는 멤버변수를 선언해주고 **protected** 접근 제어자를 통해 해당 클래스 내부 및 상속받은 하위 클래스에서만 사용가능하게 선언해 주었다.

이후 생성자에서는 baseURL을 인수로 받아 상속받은 클래스마다 다른 URL주소를 가질 수 있게 생성했다.

axios로 생성된 인스턴스의 요청 인터셉터와 응답 인터셉터를 생성해주어 반복되는 작업을 피할 수 있게 해주었다.

이제 각 필요한 도메인마다 해당 Service 클래스를 상속받아 필요한 api 요청을 작성해보도록 하자.

```tsx
import Service from "@/app/_service/index";

class UserService extends Service {
  constructor() {
    super(`BASE_URL/${USER}`); // 기본 URL을 전달
  }
  getUsers() {
    return this.http.get("/list");
  }

  getUser(userId: number) {
    return this.http.get(`/${userId}`);
  }
}
export default new UserService();
```

현재 필요한 BASE_URL을 생성자 함수에 인자로 전달해주었다. 그리고 클래스 내부에 메서드를 생성해서 필요한 엔드포인트에 연결해주어 서버에 API를 요청해주는 함수들을 클래스를 이용해서 작성해 주었다.

```tsx
import UserService from "./user-service";

const queryKeys = {
  all: ["users"],
  user: (userId: number) => [...queryKeys.all, userId] as const,
};

const queryOptions = {
  all: () => ({
    queryKey: queryKeys.all,
    queryFn: () => UserService.getUsers(),
  }),
  user: (userId: number) => ({
    queryKey: queryKeys.all,
    queryFn: () => UserService.getUser(userId),
  }),
};

export default queryOptions;
```

이후 각 쿼리키들과 쿼리 옵션들을 해당 구조에 맞게 저장해주었다.

여기서 user 쿼리키에서 첫번째 인덱스 값으로 `...queryKeys.all` 해당 값을 넣어주는 이유는 두번 째 인자인 userId가 users에 속한다는 것을 보여주기 위해 즉 동일한 쿼리키의 중복을 방지하기 위해서이다.

예를들어 userId와 나중에 productId 라는 값이 겹칠 수 도 있을 가능성도 있기 때문에 네임스페이싱을 통해서 이러한 문제를 사전에 방지하는 것이다.

여기 까지 작업해주면 서버에서 사용할 준비는 다 됐다. 하지만 클라이언트에서 빠르고 쉽게 사용하기 위해 각각의 쿼리옵션을 사용해주는 useQuery를 리턴해주는 함수들을 생성해보도록 하자

```tsx
import { useQuery } from "@tanstack/react-query";
import queryOptions from "@/app/_service/user/user-quries";

export const useUsers = () => useQuery(queryOptions.all());
export const useUser = (userId: number) => useQuery(queryOptions.user(userId));
```

이렇게 작성해주면 클라이언트에서 빠르게 useQuery를 사용할 수 있다.

## 서버에서 prefetching 작업 후 데이터 de/hydration 하기

이제 서버에서 받은 데이터를 dehydrate하고 dehydrate한 데이터를 hydrate할 수 있는 함수들을 생성하도록 하자

```tsx
import {
  HydrationBoundary,
  QueryClient,
  dehydrate,
  QueryState,
  QueryKey,
} from "@tanstack/react-query";
import { cache } from "react";
import { isEqual } from "@/app/_utils/index";

type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
interface QueryProps<ResponseType = unknown> {
  queryKey: QueryKey;
  queryFn: () => Promise<ResponseType>;
}
interface DehydratedQueryExtended<TData = unknown, TError = unknown> {
  state: QueryState<TData, TError>;
}

export const getQueryClient = cache(() => new QueryClient());

export async function getDehydratedQuery<Q extends QueryProps>({
  queryKey,
  queryFn,
}: Q) {
  const queryClient = getQueryClient();

  await queryClient.prefetchQuery({ queryKey, queryFn });

  const { queries } = dehydrate(queryClient);

  const [dehydratedQuery] = queries.filter((query) =>
    isEqual(query.queryKey, queryKey)
  );

  return dehydratedQuery as DehydratedQueryExtended<
    UnwrapPromise<ReturnType<Q["queryFn"]>>
  >;
}

export const Hydrate = HydrationBoundary;
```

해당 함수들과 타입들이 어느정도 복잡한 부분이 있으므로 하나씩 분석해보도록 하자

```tsx
export const getQueryClient = cache(() => new QueryClient());
```

- `cache` 함수를 사용하여 `QueryClient` 인스턴스가 메모이제이션되므로, 여러 번 호출해도 동일한 인스턴스가 반환되게 한다.

```tsx
export async function getDehydratedQuery<Q extends QueryProps>({
  queryKey,
  queryFn,
}: Q) {
  const queryClient = getQueryClient();

  await queryClient.prefetchQuery({ queryKey, queryFn });

  const { queries } = dehydrate(queryClient);

  const [dehydratedQuery] = queries.filter((query) =>
    isEqual(query.queryKey, queryKey)
  );

  return dehydratedQuery as DehydratedQueryExtended<
    UnwrapPromise<ReturnType<Q["queryFn"]>>
  >;
}

export const Hydrate = HydrationBoundary;
```

- `getDehydratedQuery` 함수는 `queryKey`와 `queryFn`을 받아서 미리 쿼리를 실행하고, 해당 쿼리의 결과를 디하이드레이션(dehydration)한다.
- `queryClient.prefetchQuery`를 사용하여 쿼리를 서버에서 미리 실행한다.
- `dehydrate(queryClient)`는 쿼리 클라이언트의 상태를 디하이드레이션 한다.
- 디하이드레이션된 쿼리 중에서 `queryKey`와 일치하는 쿼리를 필터링하여 반환한다.
- 그리고 마지막으로 `HydrationBoundary` 컴포넌트를 `Hydrate` 변수에 할당해 export 해준다

이제 마지막으로 서버에서 렌더링되는 함수에서 (page.tsx) queryKey와 queryFn을 불러와서 `getDehydratedQuery` 함수에 인자로 넣어주고 `Hydrate` 컴포넌트를 이용해서 children으로 client 컴포넌트를 넣어주면 된다. 아래 코드를 살펴보자

```tsx
import Userist from "@/component/user-list";
import queryOptions from "@/service/user/user-quries";
import { Hydrate, getDehydratedQuery } from "@/utils/react-query";

export default async function Home() {
  const { queryKey, queryFn } = queryOptions.all();

  const query = await getDehydratedQuery({ queryKey, queryFn });

  return (
    <main className="flex min-h-screen flex-col items-center justify-between p-24">
      <Hydrate state={{ queries: [query] }}>
        <Userist />
      </Hydrate>
    </main>
  );
}
```

이제 문제없이 서버에서 데이터를 받아와서 클라이언트에 뿌려줄 것이다.

**하지만……….**

무슨 이유인지 모르겠지만

![Untitled](<./image/tanstack(react)-query/0.png>)

해당 에러가 계속해서 발생했다. 이유를 찾아보니까 생각보다 단순한 이유였는데

```tsx
interface QueryState<TData = unknown, TError = DefaultError> {
  data: TData | undefined;
  dataUpdateCount: number;
  dataUpdatedAt: number;
  error: TError | null;
  errorUpdateCount: number;
  errorUpdatedAt: number;
  fetchFailureCount: number;
  fetchFailureReason: TError | null;
  fetchMeta: FetchMeta | null;
  isInvalidated: boolean;
  status: QueryStatus;
  fetchStatus: FetchStatus;
}
```

`getDehydratedQuery` 함수의 리턴값인 `dehydratedQuery`은 `DehydratedQueryExtended` 타입으로 지정해 줬는데 해당 타입의 **state** 프로퍼티 타입은 위의 `QueryState` 인터페이스이다. 그런데 data 프로퍼티타입을보면 TData 타입으로 지정돼있다. 해당부분은 `queryFn` 의 리턴값이 들어가는 자리인데 나는 Service 클래스에서 해당부분을 axios로 제작해서 return 값이 `AxiosResponse` 타입으로 들어가고 있었다.

```tsx
this.http.interceptors.response.use(
  (response: AxiosResponse) => {
    // 응답 데이터를 가공하거나 로깅할 수 있습니다.
    return response;
  },
  (error) => {
    // 응답 오류가 있는 작업 수행
    return Promise.reject(error);
  }
);
```

```tsx
data: {
      status: 200,
      statusText: 'OK',
      headers: [Object [AxiosHeaders]],
      config: [Object],
      request: [ClientRequest],
      data: [Array]
    },
```

그래서 해당부분을 실제 서버가 던져주는 값인 `response.data` 로 수정했다.

그 결과

```tsx
dehydratedQuery {
  state: {
    data: [ [Object] ],
    dataUpdateCount: 1,
    dataUpdatedAt: 1719936553930,
    error: null,
    errorUpdateCount: 0,
    errorUpdatedAt: 0,
    fetchFailureCount: 0,
    fetchFailureReason: null,
    fetchMeta: null,
    isInvalidated: false,
    status: 'success',
    fetchStatus: 'idle'
  },
  queryKey: [ 'users' ],
  queryHash: '["users"]'
}
```

해당 결과값을 잘 받을 수 있었고 오류도 해결했다.
