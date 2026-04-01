# Using Handle in React Router

> `handle` export + `useMatches` 훅을 조합해서 라우트 계층 구조를 기반으로 동적인 UI 요소를 만드는 패턴

---

## 핵심 개념

React Router는 현재 매칭된 **모든 라우트 정보**를 컴포넌트 트리 전반에서 사용할 수 있게 해준다.

- 각 라우트는 `handle` export로 **메타데이터**를 제공할 수 있음
- 상위 컴포넌트(ex. `root.tsx`)는 `useMatches`로 이 메타데이터를 **수집해서 렌더링**할 수 있음
- breadcrumb이 대표적인 예시지만, **어떤 패턴에도 응용 가능**

---

## 적용 흐름

```
라우트 정의 (routes.ts)
  └─ parent route → handle.breadcrumb 정의
      └─ child route → handle.breadcrumb 정의
            ↓
root.tsx → useMatches()로 모든 매치 수집 → handle.breadcrumb 렌더링
```

---

## Step 1. 라우트 구조 정의

```ts
// routes.ts
import { route } from "@react-router/dev/routes";

export default [
  route("parent", "./routes/parent.tsx", [
    route("child", "./routes/child.tsx"),
  ]),
] satisfies RouteConfig;
```

---

## Step 2. 각 라우트에 handle export 추가

```tsx
// routes/parent.tsx
import { Link } from "react-router";

export const handle = {
  breadcrumb: () => <Link to="/parent">Parent Route</Link>,
};
```

```tsx
// routes/child.tsx
import { Link } from "react-router";

export const handle = {
  breadcrumb: () => <Link to="/parent/child">Child Route</Link>,
};
```

- `handle`은 그냥 **평범한 객체**다. 프로퍼티 이름(`breadcrumb`)은 자유롭게 정하면 됨
- 함수로 정의하면 나중에 `match` 객체를 인자로 받을 수 있음

---

## Step 3. root.tsx에서 useMatches로 렌더링

```tsx
// app/root.tsx
import { Links, Meta, Outlet, Scripts, ScrollRestoration, useMatches } from "react-router";

export function Layout({ children }) {
  const matches = useMatches();

  return (
    <html lang="en">
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        <header>
          <ol>
            {matches
              .filter((match) => match.handle && match.handle.breadcrumb)
              .map((match, index) => (
                <li key={index}>
                  {match.handle.breadcrumb(match)}  {/* match 객체 전달 */}
                </li>
              ))}
          </ol>
        </header>

        {children}

        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}

export default function App() {
  return <Outlet />;
}
```

---

## 포인트 정리

| 항목 | 내용 |
|------|------|
| `handle` | 라우트에서 export하는 임의 객체. 어떤 데이터든 넣을 수 있음 |
| `useMatches` | 현재 URL에 매칭된 모든 라우트의 배열을 반환 |
| `match.handle` | 해당 라우트에서 export한 `handle` 객체 |
| `match.data` | 해당 라우트 loader에서 가져온 데이터 |
| `match.params` | URL 파라미터 |

---

## 활용 아이디어

- **Breadcrumb** - 가장 대표적인 사례
- **페이지 타이틀** - 각 라우트에서 title을 정의하고 `<head>`에 주입
- **탭/사이드바 활성 상태** - 현재 경로 기반으로 활성 메뉴 표시
- **권한/역할 메타데이터** - 각 라우트에 접근 권한 정보를 담아 상위에서 체크
- **레이아웃 분기** - 특정 라우트에서만 다른 레이아웃 적용

---

## 관련 API (React Router 공식)

- `useMatches` - 현재 매칭된 라우트 목록 반환
- `useLoaderData` - 현재 라우트의 loader 데이터
- `useRouteLoaderData` - 특정 라우트 ID의 loader 데이터
- `useParams` - URL 파라미터
- `useLocation` - 현재 location 객체

---

## 주의사항

- `handle`은 **서버/클라이언트 모두에서 정적으로 평가**된다. loader처럼 비동기가 아님
- `useMatches`는 **Framework Mode** 또는 **Data Mode**에서만 사용 가능 (Declarative Mode X)
- 각 `match.handle.breadcrumb(match)`처럼 `match` 객체를 인자로 넘기면 `match.data`를 활용한 **동적 breadcrumb**도 만들 수 있음

```tsx
// 예: loader 데이터 기반 동적 breadcrumb
export const handle = {
  breadcrumb: (match) => (
    <Link to={`/users/${match.params.id}`}>
      {match.data.user.name}  {/* loader에서 가져온 유저 이름 */}
    </Link>
  ),
};
```
