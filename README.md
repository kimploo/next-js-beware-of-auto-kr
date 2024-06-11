# nextjs-beware-of-auto-kr

Next.js 버전 별 렌더링 및 cache 관련 차이점을 정리한 레포지토리

Special Thanks To Jack Herrington
Forked and Inspired by: https://www.youtube.com/watch?v=bU3Px8bHjPA&t=178

## pages-demo

`./pages-demo` 디렉터리에는 Next.js 페이지 라우터 방식(`<=12`)에 대한 설명이 짧게 되어있다.
디펜던시 설치 (`pnpm install`) 후 빌드 (`pnpm run build`) 하면 아래 코드와 같이 각 라우트의 렌더링 방식을 보여준다.

```bash
Route (pages)                             Size     First Load JS
┌ ○ /                                     285 B          78.6 kB 
├   /_app                                 0 B            78.3 kB 
├ ○ /404                                  180 B          78.5 kB # Static
├ ƒ /dynamic                              381 B          78.7 kB # SSR (Server Side Rendering)
└ ● /static                               383 B          78.7 kB # SSG (Static Site Generation)
+ First Load JS shared by all             79.9 kB
  ├ chunks/framework-a85322f027b40e20.js  45.2 kB
  ├ chunks/main-4e0aa94d8ebb6263.js       32 kB
  └ other shared chunks (total)           2.74 kB

○  (Static)   prerendered as static content
●  (SSG)      prerendered as static HTML (uses getStaticProps)
ƒ  (Dynamic)  server-rendered on demand
```

### Static Content

Static contents는 말 그대로 콘텐츠를 따로 빌드하지 않고 보여준다.
React Component를 HTML로 렌더링하는 최소한의 prerendering 과정만 필요하기 때문에 가장 빠르지만
페이지가 동적으로 변하지 않기 때문에 interaction이 필요한 경우 부적절하다. [Documentation](https://nextjs.org/docs/pages/building-your-application/rendering/automatic-static-optimization)

### SSG (Static Site Generation)

SSG는 배포를 위해 빌드할 때, `getStaticProps()`에서 데이터를 `fetch`하고 이를 포함하여 렌더링한다.
즉, 클라이언트가 아무리 요청을 새로 보내도 빌드 시점의 데이터만 확인할 수 있다. [Documentation](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)

### Server Side Rendering

SSR은 Next.js 배포 서버 런타임에서 데이터를 `fetch`하고 이를 포함하여 렌더링한다.
즉, 클라이언트가 새롭게 요청을 보내면 그떄마다 새롭게 렌더링을 하여 데이터를 보내준다. [Documentation](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)

## app-router-demo

Next.js 13 버전 이후 사용 가능한 App 라우터 방식 예시다.
App 라우터에서는 React Server Component 및 케싱이 적용되어 개선된 점도 많지만,
새로 배우는 사람 입장에서는 조금 배우기 어려운 면도 있는데, 이 점을 정리하고자 한다.
디펜던시 설치 (`pnpm install`) 후 빌드 (`pnpm run build`) 하면 아래 코드와 같이 각 라우트의 렌더링 방식을 보여준다.

```bash
Route (app)                              Size     First Load JS
┌ ○ /                                    145 B          87.1 kB
├ ○ /_not-found                          873 B          87.8 kB
├ ƒ /dynamic                             145 B          87.1 kB
└ ○ /static                              145 B          87.1 kB
+ First Load JS shared by all            87 kB
  ├ chunks/563-4106894ffa472fbc.js       31.5 kB
  ├ chunks/669fb589-f699b5aea03e1e80.js  53.6 kB
  └ other shared chunks (total)          1.86 kB


○  (Static)   prerendered as static content
ƒ  (Dynamic)  server-rendered on demand
```

### Static

`./static` 디렉터리에 있는 페이지는 빌드 시점에 렌더링된다.
Next.js App Router의 fetch는 케싱이 default로 적용되어 있기 때문에, 아래의 코드만 작성해도 요청을 새로 보내지 않는다.

```jsx
export default async function StaticPage() {
  const req = await fetch("http://localhost:8080/api/time");
  const { time } = await req.json();
  const server = new Date().toTimeString();

  return (
    <main className="flex flex-col gap-5">
      <div className="font-bold">App Router - Static Page</div>
      <div>Server Time: {server}</div>
      <div>API Time: {time}</div>
    </main>
  );
}
```

### Dynamic

`./dynamic` 디렉터리에 있는 페이지는 유저 요청 시점에 렌더링된다.
유저가 요청할 때 마다 이 부분을 렌더링시키기 위해서는 `dynamic` 변수에 `"force-dynamic"`을 적용하고 `fetch`의 cache 옵션을 사용하면 된다.
Next.js 14 버전까지는 이 옵션을 둘 다 적용해야 한다. `fetch`를 Next.js에서 커스텀하여 케싱을 자동으로 적용해왔기 때문이다.
원성이 자자한 부분이라 Next.js 15 버전에서는 디폴트 케싱을 `fetch`에서 해제한다고 한다.

```jsx
// #2
export const dynamic = "force-dynamic";

export default async function DynamicPage() {
  // #1
  const req = await fetch("http://localhost:8080/api/time", {
    cache: "no-store",
  });
  const { time } = await req.json();

  const server = new Date().toTimeString();

  return (
    <main className="flex flex-col gap-5">
      <div className="font-bold">App Router - Dynamic Page</div>
      <div>Server Time: {server}</div>
      <div>API Time: {time}</div>
    </main>
  );
}
```

### Client Component

Next.js의 Client Component는 예시에 포함되어있지는 않아 정리한다.
컴포넌트 상단에 `'use client'`라고 적으면 아래 컴포넌트는 기존에 React Function Component처럼 여러 hook을 사용할 수 있다.
Next.js에서는 빠른 렌더링을 위해 서버에서 먼저 HTML을 렌더링하여 브라우저에 전달하고,
`The React Server Components Payload`가 클라이언트, 서버 컴포넌트 트리 재조정 및 DOM 업데이트를 거친다
이후 "hydrate"을 위한 JavaScript가 HTML에 적용된다.

조금의 논리 비약을 거치면 아래와 같이 요약할 수 있다.
서버에서 HTML이 먼저 렌더링되고, 이후에 클라이언트에 꼭 필요한 payload, JavaScript가 전달되어 상호작용이 가능한 웹앱을 완성한다.
[Documentation](https://nextjs.org/docs/app/building-your-application/rendering/client-components#how-are-client-components-rendered)

```jsx
'use client'
 
import { useState } from 'react'
 
export default function Counter() {
  const [count, setCount] = useState(0)
 
  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

## IMO, Lesson Learned

- Page Router는 페이지 단위로 렌더링 방식을 결정할 수 있었지만, App Router에서는 어떤 React Component든 렌더링 방식을 정할 수 있게 되었다.
- App Router에서는 SWR이나 React Query처럼 케싱 기능도 제공하기 위해서 케싱을 디폴트로 적용했다.
  - 의도는 좋았으나, 코드의 작동을 명시적으로 확인할 수 없었기에 많은 개발자가 케싱 현상에 대해 불만을 가졌고 Next.js 15 버전에서 변경될 예정이다.
  - Vercel과 같이 React 개발진과 협업하는 소프트웨어 기업에서도 이런 실수를 하는구나 싶다.
  - 개인적으로 `dynamic` 변수나 `'use client`'와 같은 문법은 타입 추론이 가능했으면 좋겠다.
  - 나도 나만 알아볼 수 있는 코드를 적었던 기억이 있는데, 누구든 이해할 수 있게 명확한 타입 지정과 문서화를 해야겠다고 생각했다.
- 개인적으로 Next.js 공식 문서는 다소 읽기 어려웠던 면이 있다.
  - Documentation과 API 명세가 섞여있어 여러번 읽고 유튜브 검색도 해봐야 했다.
  - ChatGPT도 최신 문법에 대해서 잘 알려주진 않더라.
  - 이런 정보를 잘 이해하는 것도 능력인 것 같고 앞으로 잘 정리해야겠다.
  - 깔끔하게 설명해준 Jack Herrington 형님에게 박수 👏