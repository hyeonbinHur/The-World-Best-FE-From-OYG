# SPA (Single Page App) in React Router

> Framework Mode 전용 가이드. Declarative/Data Mode는 자체 SPA 아키텍처 설계 가능.

---

## SPA Mode란?

`react-router.config.ts`에서 `ssr: false` 설정 하나로 SPA Mode 활성화.

- 런타임 서버 렌더링 비활성화
- 빌드 시점에 `index.html` 생성 → 이걸 SPA로 서빙 + hydrate

### 일반 SPA vs React Router SPA Mode

| 일반 SPA | React Router SPA Mode |
|---|---|
| 빈 `<div id="root"></div>` 전송 | 빌드 시점에 루트 라우트를 pre-render |
| 데이터 없이 시작 | 루트 `loader`로 앱 셸 데이터 로딩 가능 |
| 커스텀 로딩 UI 구현 복잡 | `HydrateFallback` 컴포넌트로 초기 로딩 UI 제공 |

> SPA Mode = 모든 경로를 같은 `index.html`로 서빙하는 특수한 Pre-Rendering 형태.
> 더 광범위한 pre-rendering이 필요하면 Pre-Rendering 가이드 참고.

---

## 세팅 순서

### 1. 런타임 서버 렌더링 비활성화

```ts
// react-router.config.ts
import { type Config } from "@react-router/dev/config";

export default {
  ssr: false,
} satisfies Config;
```

**주의할 점:**
- `ssr: false` = 런타임 서버 렌더링만 꺼짐
- 빌드 시점에 `index.html` 생성을 위해 루트 라우트는 여전히 서버 렌더링됨
- 그래서 `@react-router/node` 의존성은 여전히 필요
- 초기 렌더링 중 `window` 같은 브라우저 전용 API 사용 불가 (SSR-safe 유지 필수)

---

### 2. 루트 라우트에 `HydrateFallback` + (선택) `loader` 추가

SPA Mode는 빌드 시 루트 라우트만 렌더링해서 `index.html`을 만듦.
모든 경로의 런타임 hydration 진입점이 되는 파일.

#### HydrateFallback - 로딩 중 보여줄 UI

```tsx
import LoadingScreen from "./components/loading-screen";

export function Layout() {
  return <html>{/* ... */}</html>;
}

// 빌드 시점에 index.html에 렌더링 → SPA 로딩/hydrate 중 즉시 표시됨
export function HydrateFallback() {
  return <LoadingScreen />;
}

export default function App() {
  return <Outlet />;
}
```

#### (선택) 루트 loader로 빌드 시점 데이터 주입

루트 라우트는 빌드 시 서버 렌더링되므로 `loader` 사용 가능.
이 `loader`는 **빌드 시점에 호출**되고, 데이터는 `HydrateFallback`의 `loaderData`로 전달.

```tsx
import { Route } from "./+types/root";

export async function loader() {
  return {
    version: await getVersion(),
  };
}

export function HydrateFallback({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      <h1>Loading version {loaderData.version}...</h1>
      <AwesomeSpinner />
    </div>
  );
}
```

> 다른 라우트에는 `loader` 사용 불가 (pre-rendering하는 경우 제외).

---

### 3. `clientLoader` / `clientAction` 사용

서버 렌더링이 꺼져 있어도 라우트 데이터 로딩과 뮤테이션은 이걸로 처리.

```tsx
import { Route } from "./+types/some-route";

// 데이터 로딩
export async function clientLoader({ params }: Route.ClientLoaderArgs) {
  let data = await fetch(`/some/api/stuff/${params.id}`);
  return data;
}

// 뮤테이션 (form submit 등)
export async function clientAction({ request }: Route.ClientActionArgs) {
  let formData = await request.formData();
  return await processPayment(formData);
}
```

---

### 4. 모든 URL을 `index.html`로 연결

```bash
react-router build
# → build/client 디렉터리 생성
```

정적 호스팅에 `build/client` 배포 후, **모든 URL 요청이 `index.html`로 향하도록** 호스트 설정 필요.

```txt
# _redirects 파일 예시 (Netlify 등)
/*    /index.html   200
```

> 유효한 라우트에서 404가 발생하면 → 호스트 설정 문제일 가능성 높음.

---

## 핵심 요약

```
ssr: false 설정
    ↓
빌드 시 루트 라우트만 서버 렌더링 → index.html 생성
    ↓
클라이언트에서 hydrate (HydrateFallback이 그 사이 로딩 UI 담당)
    ↓
이후 모든 라우팅/데이터는 clientLoader / clientAction으로 처리
    ↓
배포: build/client → 정적 호스팅, 모든 URL → index.html
```

- 나중에 SSR 다시 켜도 UI 변경 없음 (`ssr: true`만 되돌리면 됨)
- SPA Mode = Pre-Rendering의 특수 형태 (루트만 pre-render)
