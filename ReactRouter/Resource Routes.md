# Resource Routes

> 서버 렌더링 시, 라우트는 컴포넌트를 렌더링하는 대신 이미지, PDF, JSON payload, webhook 같은 **"resource"** 를 제공할 수 있다.

---

## 1. Defining a Resource Route (리소스 라우트 정의)

**핵심 규칙:** `loader` 또는 `action`을 export하고, **`default` 컴포넌트를 export하지 않으면** 해당 라우트는 자동으로 리소스 라우트가 된다.

```ts
// routes.ts
route("/reports/pdf/:id", "pdf-report.ts");
```

```ts
// pdf-report.ts
import type { Route } from "./+types/pdf-report";

export async function loader({ params }: Route.LoaderArgs) {
  const report = await getReport(params.id);
  const pdf = await generateReportPDF(report);
  return new Response(pdf, {
    status: 200,
    headers: {
      "Content-Type": "application/pdf",
    },
  });
}
// default export 없음 → 리소스 라우트!
```

> `default export`가 없다는 게 포인트. 컴포넌트가 없으니 렌더링 대신 데이터/파일을 직접 반환한다.

---

## 2. Linking to Resource Routes (리소스 라우트로 연결하기)

리소스 라우트로 링크할 때는 반드시 `<a>` 또는 `<Link reloadDocument>`를 사용해야 한다.

```tsx
<Link reloadDocument to="/reports/pdf/123">
  View as PDF
</Link>
```

**왜?** 일반 `<Link>`를 쓰면 React Router가 클라이언트 사이드 라우팅으로 처리하려 해서, payload를 가져오려다 에러가 난다.

| 방법 | 동작 |
|---|---|
| `<a href="...">` | 브라우저가 직접 요청 → OK |
| `<Link reloadDocument>` | 풀 페이지 로드 → OK |
| `<Link>` (일반) | 클라이언트 사이드 라우팅 시도 → 에러 |

---

## 3. Handling Different Request Methods (HTTP 메서드 분기)

| HTTP Method | 처리 함수 |
|---|---|
| `GET` | `loader` |
| `POST` / `PUT` / `PATCH` / `DELETE` | `action` |

```ts
import type { Route } from "./+types/resource";

export function loader(_: Route.LoaderArgs) {
  return Response.json({ message: "I handle GET" });
}

export function action(_: Route.ActionArgs) {
  return Response.json({
    message: "I handle everything else", // POST, PUT, PATCH, DELETE
  });
}
```

---

## 4. Return Types (반환 타입)

리소스 라우트에서 `Response` 인스턴스와 `data()` 객체 중 무엇을 반환할지는 **사용 목적**에 따라 다르다.

| 상황 | 권장 반환 타입 | 이유 |
|---|---|---|
| 외부 클라이언트(브라우저, 앱 등)가 직접 소비 | `new Response()` | 응답 인코딩을 코드에서 명시적으로 관리 가능 |
| `fetcher` 또는 `<Form>` 제출에서 접근 | `data()` | UI 라우트의 loader/action과 일관성 유지, `Await`로 스트리밍 가능 |

---

## 5. Error Handling (에러 처리)

### Fatal Error → 500

`Error` 객체를 throw하거나 `Response`/`data()` 이외의 값을 throw하면 `handleError`가 트리거되고 **500 응답**이 반환된다.

```ts
export function action() {
  let db = await getDb();
  if (!db) {
    // handleError 트리거 → 500
    throw new Error("Could not connect to DB");
  }
}
```

### Non-Fatal Error → 4xx/5xx Response

`Response`를 생성하면 (throw든 return이든) **성공적인 실행**으로 간주하고 `handleError`를 트리거하지 않는다.

```ts
export function action() {
  // 아래 4가지는 모두 동일하게 동작 (handleError 미트리거)

  throw new Response({ error: "Unauthorized" }, { status: 401 });

  return new Response({ error: "Unauthorized" }, { status: 401 });

  throw data({ error: "Unauthorized" }, { status: 401 });

  return data({ error: "Unauthorized" }, { status: 401 });
}
```

> `fetch()`가 4xx/5xx에 대해 rejected promise를 반환하지 않는 것과 같은 맥락이다.

**정리:**
- `throw new Error(...)` → **Fatal** → `handleError` → 500
- `throw new Response(...)` / `return new Response(...)` → **Non-Fatal** → `handleError` 없음

### 5.1 Error Boundaries

`Error Boundaries`는 리소스 라우트가 **UI에서 접근될 때**만 적용된다.

- `fetcher` 호출이나 `<Form>` 제출에서 리소스 라우트가 throw하면 → UI의 가장 가까운 `ErrorBoundary`로 버블링
- 직접 URL로 접근하는 경우 → ErrorBoundary 적용 안 됨

---

## 핵심 요약

```
default export 없음 = 리소스 라우트
GET → loader / 나머지 → action
외부용 → Response / fetcher·Form용 → data()
Error throw → 500 / Response throw → Non-Fatal
```
