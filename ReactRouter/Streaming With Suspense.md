# Streaming with Suspense

> React Suspense를 이용한 스트리밍 = 중요하지 않은 데이터를 뒤로 미뤄서 **초기 렌더링을 더 빠르게** 하고, UI가 막히지 않도록 하는 기법

React Router는 `loader`와 `action`에서 Promise를 반환하는 방식으로 React Suspense를 지원한다.

---

## 핵심 아이디어

- 보통 React Router는 route loader가 **전부 완료**된 후에야 컴포넌트를 렌더링한다
- 근데 중요하지 않은 데이터가 느릴 때, 그걸 기다리느라 전체 페이지가 블로킹되면 UX가 나빠진다
- 해결책: 느린 데이터는 **await 하지 말고 Promise 그대로** 반환 → 컴포넌트에서 Suspense로 처리

---

## 1. loader에서 Promise 반환하기

```tsx
import type { Route } from "./+types/my-route";

export async function loader({}: Route.LoaderArgs) {
  // await 안 함 → Promise 자체를 넘긴다
  const nonCriticalData = new Promise((resolve) =>
    setTimeout(() => resolve("non-critical"), 5000),
  );

  // 중요한 데이터는 await
  const criticalData = await new Promise((resolve) =>
    setTimeout(() => resolve("critical"), 300),
  );

  return { nonCriticalData, criticalData };
}
```

> **주의**: Promise 하나만 단독으로 반환할 수 없다. 반드시 key가 있는 **객체 형태**로 반환해야 한다.

---

## 2. fallback UI와 완료된 UI 렌더링하기

`loaderData`에서 받은 Promise를 `<Await>`에 넘기면,
Promise가 끝날 때까지 `<Suspense>`가 fallback UI를 보여준다.

```tsx
import * as React from "react";
import { Await } from "react-router";

export default function MyComponent({ loaderData }: Route.ComponentProps) {
  const { criticalData, nonCriticalData } = loaderData;

  return (
    <div>
      <h1>Streaming example</h1>

      {/* 중요한 데이터는 바로 렌더링 */}
      <h2>Critical data: {criticalData}</h2>

      {/* 중요하지 않은 데이터는 Suspense로 감싸서 스트리밍 */}
      <React.Suspense fallback={<div>Loading...</div>}>
        <Await resolve={nonCriticalData}>
          {(value) => <h3>Non critical value: {value}</h3>}
        </Await>
      </React.Suspense>
    </div>
  );
}
```

### 흐름 정리

```
loader 실행
  └─ criticalData → await → 완료 후 렌더링 시작
  └─ nonCriticalData → Promise 그대로 반환

컴포넌트 렌더링
  └─ criticalData → 즉시 표시
  └─ <Suspense fallback> → "Loading..." 표시
       └─ Promise 완료되면 → <Await> 내부 UI로 교체
```

---

## 3. React 19에서 사용하는 방법

React 19에서는 `<Await>` 대신 `React.use()`를 쓸 수 있다.
단, Suspense fallback이 동작하려면 **Promise를 받는 별도 컴포넌트**로 분리해야 한다.

```tsx
<React.Suspense fallback={<div>Loading...</div>}>
  <NonCriticalUI p={nonCriticalData} />
</React.Suspense>
```

```tsx
function NonCriticalUI({ p }: { p: Promise<string> }) {
  const value = React.use(p);
  return <h3>Non critical value: {value}</h3>;
}
```

> `React.use(p)`는 Promise가 resolve될 때까지 컴포넌트를 suspend시킨다.
> 부모의 `<Suspense>` fallback이 그 동안 대신 렌더링된다.

### `<Await>` vs `React.use` 비교

| | `<Await>` (React Router) | `React.use` (React 19) |
|---|---|---|
| 방식 | render prop 패턴 | hook |
| 별도 컴포넌트 분리 | 불필요 | 필요 |
| 안정성 | 안정 | 실험적 |

---

## 4. Timeouts

- 기본값: loader/action에서 아직 끝나지 않은 Promise는 **4950ms 후 자동으로 reject**된다
- `entry.server.tsx`에서 `streamTimeout` 값을 export해서 조절 가능

```ts
// entry.server.tsx
// handler 함수에서 보류 중인 Promise를 10초 후 모두 reject
export const streamTimeout = 10_000;
```

> timeout 설정은 서버 사이드에서만 적용되는 설정임을 기억하자.

---

## 핵심 요약

1. **느린 데이터** → `await` 없이 Promise 그대로 반환
2. **빠른 데이터** → 평소처럼 `await`
3. 컴포넌트에서 `<Suspense>` + `<Await>`로 감싸기
4. React 19라면 `React.use()`로 대체 가능 (단, 별도 컴포넌트로 분리 필요)
5. timeout 기본값은 4950ms, `entry.server.tsx`에서 조절 가능
