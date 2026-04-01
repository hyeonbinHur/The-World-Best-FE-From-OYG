# React Router - React Server Components (RSC) 필기

> **주의**: RSC 지원은 실험적(unstable)이며, 마이너/패치 릴리스에서도 breaking change가 발생할 수 있음. 릴리스 노트를 꼼꼼히 확인할 것.

---

## RSC란?

React 19부터 제공되는 아키텍처 + API 집합.

> "Server Components are a new type of Component that renders **ahead of time**, before bundling, in an environment separate from your client app or SSR server."

핵심: 번들링 이전에, 클라이언트 앱과 분리된 환경에서 미리 렌더링되는 컴포넌트.

React Router는 RSC 호환 번들러와 통합하기 위한 API를 제공 → React Router 앱에서 **Server Components + Server Functions** 사용 가능.

RSC 지원은 **Framework Mode**와 **Data Mode** 모두에서 사용 가능.

---

## 1. Quick Start

템플릿으로 빠르게 시작 가능. 이미 RSC API가 설정되어 있음.

기본 제공 기능:
- Server Component Routes
- SSR (Server Side Rendering)
- Client Components (`"use client"` 지시어)
- Server Functions (`"use server"` 지시어)

### Framework Mode 템플릿

`unstable_reactRouterRSC` Vite 플러그인 + `@vitejs/plugin-rsc` 플러그인 사용.

```shell
npx create-react-router@latest --template remix-run/react-router-templates/unstable_rsc-framework-mode
```

### Data Mode 템플릿

```shell
npx create-react-router@latest --template remix-run/react-router-templates/unstable_rsc-data-mode-vite
```

---

## 2. RSC Framework Mode

비 RSC Framework Mode와 대부분 동일 → **차이점에 집중**.

### 2.1. Vite 플러그인

기존 플러그인 대신 `unstable_reactRouterRSC` 사용.
`@vitejs/plugin-rsc`는 peer dependency이며, **React Router RSC 플러그인 뒤에** 위치해야 함.

```tsx
import { defineConfig } from "vite";
import { unstable_reactRouterRSC as reactRouterRSC } from "@react-router/dev/vite";
import rsc from "@vitejs/plugin-rsc";

export default defineConfig({
  plugins: [reactRouterRSC(), rsc()], // 순서 중요!
});
```

### 2.2. Build Output

서버 빌드 파일(`build/server/index.js`)이 이제 `default` 요청 핸들러 함수를 export함.

```
(request: Request) => Promise<Response>
```

Express에서 사용하려면 `@remix-run/node-fetch-server`의 `createRequestListener`로 변환:

```tsx
import express from "express";
import requestHandler from "./build/server/index.js";
import { createRequestListener } from "@remix-run/node-fetch-server";

const app = express();

app.use("/assets", express.static("build/client/assets", { immutable: true, maxAge: "1y" }));
app.use(express.static("build/client"));
app.use(createRequestListener(requestHandler));
app.listen(3000);
```

### 2.3. Loader/Action에서 React Element 반환

RSC Framework Mode에서는 loader/action이 **React element도 반환 가능**.
이 element들은 **항상 서버에서만 렌더링**됨.

```tsx
export async function loader() {
  return {
    message: "Message from the server!",
    element: <p>Element from the server!</p>, // 서버에서만 렌더링
  };
}

export default function Route({ loaderData }: Route.ComponentProps) {
  return (
    <>
      <h1>{loaderData.message}</h1>
      {loaderData.element}
    </>
  );
}
```

element 안에서 Hooks, event handler 같은 client 전용 기능이 필요하면 → **`"use client"` 모듈로 분리**:

```tsx
// counter.tsx
"use client";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

```tsx
// route.tsx
import { Counter } from "./counter";

export async function loader() {
  return {
    element: (
      <>
        <p>Element from the server!</p>
        <Counter /> {/* client module */}
      </>
    ),
  };
}
```

### 2.4. Server Component Routes

라우트에서 `default` 대신 **`ServerComponent`를 export**하면 해당 라우트 전체가 Server Component가 됨.
(`ErrorBoundary`, `HydrateFallback`, `Layout`도 포함)

```tsx
export function ServerComponent({ loaderData }: Route.ComponentProps) {
  return (
    <>
      <h1>Server Component Route</h1>
      <p>Message from the server: {loaderData.message}</p>
      <Outlet />
    </>
  );
}
```

마찬가지로 client 전용 기능은 `"use client"` 모듈로 분리해야 함.

### 2.5. `.server` / `.client` 모듈

RSC Framework Mode에서는 `.server` / `.client` 파일 네이밍 규칙이 **더 이상 내장 지원되지 않음**.
→ RSC의 `"use server"` / `"use client"` 지시어와 혼동 방지.

**대안**: `@vitejs/plugin-rsc`가 제공하는 `"server-only"` / `"client-only"` import 사용.

```ts
import "server-only";
// Rest of the module...
```

> `server-only`, `client-only` npm 패키지를 직접 설치할 필요 없음.
> `@vitejs/plugin-rsc`가 내부적으로 처리하며, **runtime error 대신 build-time validation**을 제공함.

기존 `.server`/`.client` 파일 네이밍 규칙을 빠르게 마이그레이션하려면 `vite-env-only` 플러그인 사용:

```tsx
import { denyImports } from "vite-env-only";

export default defineConfig({
  plugins: [
    denyImports({
      client: { files: ["**/.server/*", "**/*.server.*"] },
    }),
    reactRouterRSC(),
    rsc(),
  ],
});
```

### 2.6. MDX Route 지원

`@mdx-js/rollup` v3.1.1+ 사용 시 MDX route 지원.
MDX route에서 export하는 컴포넌트는 RSC 환경에서 유효해야 함 → Hooks 사용 불가, client module로 분리 필요.

### 2.7. Custom Entry Files

플러그인이 `app` 디렉터리에서 자동으로 감지하는 파일:

| 파일 | 역할 |
|------|------|
| `app/entry.rsc.ts` (또는 `.tsx`) | Custom RSC server entry |
| `app/entry.ssr.ts` (또는 `.tsx`) | Custom SSR server entry |
| `app/entry.client.tsx` | Custom client entry |

없으면 React Router의 기본 entry 사용.

**기본 동작 확장 패턴 (RSC entry 커스텀 예시):**

```ts
import defaultEntry from "@react-router/dev/config/default-rsc-entries/entry.rsc";
import { RouterContextProvider } from "react-router";

export default {
  fetch(request: Request): Promise<Response> {
    console.log("Custom RSC entry handling request:", request.url);
    const requestContext = new RouterContextProvider();
    return defaultEntry.fetch(request, requestContext);
  },
};

if (import.meta.hot) {
  import.meta.hot.accept();
}
```

**필수 export 규칙:**
- `entry.rsc.ts` → `fetch` 메서드를 가진 default object export
- `entry.ssr.ts` → `generateHTML` 함수 export
- `entry.client.tsx` → client-side hydration 처리

기본 entry 위치: `node_modules/@react-router/dev/dist/config/default-rsc-entries/`

### 2.8. 아직 지원되지 않는 Config 옵션 (초기 unstable 릴리스 기준)

`react-router.config.ts`에서 RSC Framework Mode 미지원 옵션:
- `buildEnd`, `prerender`, `presets`, `routeDiscovery`, `serverBundles`
- `ssr: false` (SPA Mode)
- `future.v8_splitRouteModules`
- `future.unstable_subResourceIntegrity`

---

## 3. RSC Data Mode

RSC Framework Mode의 하위 레벨 API.
Framework Mode보다 유연하지만 일부 기능 없음:
- `routes.ts` 기반 파일 시스템 라우팅 없음
- HMR / Hot Data Revalidation 없음

직접 번들러와 서버 추상화에 통합할 때 유용.

### 3.1. 라우트 설정

라우트는 `matchRSCServerRequest`의 인자로 설정. 최소한 `path`와 `component` 필요.

```tsx
matchRSCServerRequest({
  routes: [{ path: "/", Component: Root }],
});
```

**권장 방식**: `lazy()` + Route Module 정의 (초기 성능 + 코드 구성)

```tsx
import type { unstable_RSCRouteConfig as RSCRouteConfig } from "react-router";

export function routes() {
  return [
    {
      id: "root",
      path: "",
      lazy: () => import("./root/route"),
      children: [
        { id: "home", index: true, lazy: () => import("./home/route") },
        { id: "about", path: "about", lazy: () => import("./about/route") },
      ],
    },
  ] satisfies RSCRouteConfig;
}
```

> Route Module API가 Framework Mode 전용이었는데, RSC route config의 `lazy` 필드가 동일한 export를 기대함으로써 두 API가 통합됨.

### 3.2. Server Component Routes (Data Mode)

Data Mode에서는 기본적으로 각 라우트의 `default` export가 Server Component로 렌더링됨.

```tsx
export default function Home() {
  return <main><h1>Welcome to React Router RSC</h1></main>;
}
```

Server Component의 장점: **비동기 함수**로 만들어 컴포넌트 안에서 직접 데이터 fetch 가능.

```tsx
export default async function Home() {
  let user = await getUserData(); // 컴포넌트에서 직접 fetch!
  return (
    <main>
      <h1>Welcome to React Router RSC</h1>
      <p>Hello, {user ? user.name : "anonymous person"}!</p>
    </main>
  );
}
```

- loader/action에서도 Server Component 반환 가능
- RSC 앱에서 loader는 주로 `status` code 설정이나 `redirect` 용도로 유용
- loader에서 Server Component 반환 → RSC 점진적 도입 시 유용

### 3.3. Server Functions

`"use server"` 지시어로 정의. 서버에서 실행되는 비동기 함수를 클라이언트에서 호출 가능.

```tsx
// action.ts
"use server";

export async function updateFavorite(formData: FormData) {
  let movieId = formData.get("id");
  let intent = formData.get("intent");
  if (intent === "add") {
    await addFavorite(Number(movieId));
  } else {
    await removeFavorite(Number(movieId));
  }
}
```

```tsx
// component.tsx
import { updateFavorite } from "./action.ts";

export async function AddToFavoritesForm({ movieId }: { movieId: number }) {
  let isFav = await isFavorite(movieId);
  return (
    <form action={updateFavorite}> {/* Server Function을 action으로 직접 전달 */}
      <input type="hidden" name="id" value={movieId} />
      <input type="hidden" name="intent" value={isFav ? "remove" : "add"} />
      <AddToFavoritesButton isFav={isFav} />
    </form>
  );
}
```

> Server Function 호출 후 React Router가 **자동으로 라우트를 revalidate**하고 UI 업데이트.
> Cache invalidation 따로 처리할 필요 없음!

### 3.4. Client Properties

라우트는 서버에서 정의되지만, `"use client"` 활용해 `clientLoader`, `clientAction`, `shouldRevalidate` 제공 가능.

```tsx
// route.client.tsx
"use client";

export function clientAction() {}
export function clientLoader() {}
export function shouldRevalidate() {}
```

```tsx
// route.tsx (lazy loaded route module에서 re-export)
export { clientAction, clientLoader, shouldRevalidate } from "./route.client";

export default function Root() {
  // ...
}
```

**전체 라우트를 Client Component로 만드는 방법:**

```tsx
import { default as ClientRoot } from "./route.client";
export { clientAction, clientLoader, shouldRevalidate } from "./route.client";

export default function Root() {
  // css side-effect imports를 위해 Server Component가 루트에 하나 있어야 함
  return <ClientRoot />;
}
```

### 3.5. Bundler Configuration (Entry Points)

RSC Data Mode에서 번들러 통합을 위해 3가지 entry point 설정 필요:

| Entry | 역할 |
|-------|------|
| `entry.ssr.tsx` | 서버. 요청 처리 → RSC 서버 호출 → RSC payload를 HTML로 변환 |
| `entry.rsc.tsx` | React Server. 요청을 라우트에 매칭 → RSC payload 생성 |
| `entry.browser.tsx` | 클라이언트. HTML hydrate + `callServer` 설정 |

> **포인트**: "React Server"와 SSR 서버가 물리적으로 같은 서버일 수 있음.
> 단, React가 RSC payload 생성 시와 hydrate용 HTML 생성 시 다르게 동작하므로 **2개의 module graph**가 필요.

**Relevant APIs 정리:**

- `entry.ssr.tsx`: `routeRSCServerRequest`, `RSCStaticRouter`
- `entry.rsc.tsx`: `matchRSCServerRequest`
- `entry.browser.tsx`: `createCallServer`, `getRSCStream`, `RSCHydratedRouter`

---

## 4. Vite 설정 (Data Mode 전체 예시)

### 의존성

```shell
npm i -D vite @vitejs/plugin-react @vitejs/plugin-rsc
```

### `vite.config.ts`

```ts
import rsc from "@vitejs/plugin-rsc/plugin";
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    react(),
    rsc({
      entries: {
        client: "src/entry.browser.tsx",
        rsc: "src/entry.rsc.tsx",
        ssr: "src/entry.ssr.tsx",
      },
    }),
  ],
});
```

### `entry.ssr.tsx`

```tsx
import { createFromReadableStream } from "@vitejs/plugin-rsc/ssr";
import { renderToReadableStream as renderHTMLToReadableStream } from "react-dom/server.edge";
import {
  unstable_routeRSCServerRequest as routeRSCServerRequest,
  unstable_RSCStaticRouter as RSCStaticRouter,
} from "react-router";

export async function generateHTML(request: Request, serverResponse: Response): Promise<Response> {
  return await routeRSCServerRequest({
    request,
    serverResponse,
    createFromReadableStream,
    async renderHTML(getPayload) {
      const payload = getPayload();
      const bootstrapScriptContent = await import.meta.viteRsc.loadBootstrapScriptContent("index");

      return await renderHTMLToReadableStream(
        <RSCStaticRouter getPayload={getPayload} />,
        { bootstrapScriptContent, formState: payload.formState },
      );
    },
  });
}
```

### `entry.rsc.tsx`

```tsx
import {
  createTemporaryReferenceSet,
  decodeAction,
  decodeFormState,
  decodeReply,
  loadServerAction,
  renderToReadableStream,
} from "@vitejs/plugin-rsc/rsc";
import { unstable_matchRSCServerRequest as matchRSCServerRequest } from "react-router";
import { routes } from "./routes/config";

function fetchServer(request: Request) {
  return matchRSCServerRequest({
    createTemporaryReferenceSet,
    decodeAction,
    decodeFormState,
    decodeReply,
    loadServerAction,
    request,
    routes: routes(),
    generateResponse(match) {
      return new Response(renderToReadableStream(match.payload), {
        status: match.statusCode,
        headers: match.headers,
      });
    },
  });
}

export default async function handler(request: Request) {
  const ssr = await import.meta.viteRsc.loadModule<typeof import("./entry.ssr")>("ssr", "index");
  return ssr.generateHTML(request, await fetchServer(request));
}
```

### `entry.browser.tsx`

```tsx
import {
  createFromReadableStream,
  createTemporaryReferenceSet,
  encodeReply,
  setServerCallback,
} from "@vitejs/plugin-rsc/browser";
import { startTransition, StrictMode } from "react";
import { hydrateRoot } from "react-dom/client";
import {
  unstable_createCallServer as createCallServer,
  unstable_getRSCStream as getRSCStream,
  unstable_RSCHydratedRouter as RSCHydratedRouter,
  type unstable_RSCPayload as RSCServerPayload,
} from "react-router";

// hydration 이후 server actions 지원을 위한 callServer 설정
setServerCallback(
  createCallServer({ createFromReadableStream, createTemporaryReferenceSet, encodeReply }),
);

// 초기 서버 payload 디코딩 후 hydrate
createFromReadableStream<RSCServerPayload>(getRSCStream()).then((payload) => {
  startTransition(async () => {
    const formState = payload.type === "render" ? await payload.formState : undefined;

    hydrateRoot(
      document,
      <StrictMode>
        <RSCHydratedRouter
          createFromReadableStream={createFromReadableStream}
          payload={payload}
        />
      </StrictMode>,
      { formState },
    );
  });
});
```

---

## 핵심 정리

| 구분 | Framework Mode | Data Mode |
|------|---------------|-----------|
| 파일 시스템 라우팅 | O | X |
| HMR / Hot Data Revalidation | O | X |
| 유연성 | 낮음 | 높음 |
| 번들러 커스텀 통합 | 어려움 | 가능 |
| Vite 플러그인 | `unstable_reactRouterRSC` | `@vitejs/plugin-rsc` 직접 |

**RSC에서 client 전용 기능(Hooks, event handler) 필요 시 → 항상 `"use client"` 모듈로 분리!**

**Server Function 호출 후 revalidation은 React Router가 자동 처리 → cache invalidation 불필요!**
