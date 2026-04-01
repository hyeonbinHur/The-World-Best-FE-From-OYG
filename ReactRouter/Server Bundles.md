# Server Bundles

> 호스팅 제공자 통합을 위해 설계된 고급 기능.
> 앱을 여러 서버 번들로 컴파일할 때는, **요청을 올바른 번들로 전달하는 커스텀 라우팅 계층**이 앱 앞단에 필요하다.

---

## 왜 쓰는가?

React Router는 기본적으로 서버 코드를 **하나의 번들**로 빌드하고, 그 번들이 `request handler` 함수를 export 한다.

하지만 라우트 트리를 **여러 서버 번들로 나누고**, 각 번들이 라우트 일부에 대한 request handler를 노출하고 싶을 때 사용한다.

---

## 설정 방법

`react-router.config.ts`의 `serverBundles` 옵션을 사용한다.

- 이 함수는 **라우트를 서로 다른 서버 번들에 할당**한다.
- 트리의 각 라우트에 대해 호출되며, **서버 번들 ID**를 반환한다.
- 단, pathless layout route 같이 주소 지정이 불가능한 라우트는 제외된다.
- 반환된 번들 ID는 서버 빌드 디렉터리 안에서 **디렉터리 이름**으로 사용된다.

### branch란?

각 라우트에 대해 함수는 **해당 라우트까지 이어지는 라우트 배열**을 받는데, 이를 `branch`라고 한다.

이를 통해 라우트 트리의 서로 다른 부분에 대해 서버 번들을 만들 수 있다.
예를 들어, 특정 layout route 내부의 모든 라우트를 별도 서버 번들로 분리할 수 있다.

```ts
import type { Config } from "@react-router/dev/config";

export default {
  // ...
  serverBundles: ({ branch }) => {
    const isAuthenticatedRoute = branch.some((route) =>
      route.id.split("/").includes("_authenticated"),
    );

    return isAuthenticatedRoute
      ? "authenticated"
      : "unauthenticated";
  },
} satisfies Config;
```

> `_authenticated` 세그먼트를 포함하는 라우트면 `"authenticated"` 번들,
> 아니면 `"unauthenticated"` 번들로 분류하는 예시.

---

## branch 배열의 route 속성

| 속성 | 설명 |
|------|------|
| `id` | 라우트 고유 ID. `app` 디렉터리 기준 상대 경로이며 확장자 제외. (예: `routes/gists.$username`) |
| `path` | URL pathname 과 매칭할 때 사용하는 경로 |
| `file` | 라우트 엔트리 포인트의 절대 경로 |
| `index` | index route 여부 (boolean) |

---

## Build Manifest

빌드가 완료되면 React Router는 `buildEnd` 훅을 호출하면서 `buildManifest` 객체를 전달한다.

요청을 올바른 서버 번들로 라우팅하는 방법을 결정할 때 이 객체를 확인한다.

```ts
import type { Config } from "@react-router/dev/config";

export default {
  // ...
  buildEnd: async ({ buildManifest }) => {
    // buildManifest를 이용해 번들 라우팅 로직 구성
  },
} satisfies Config;
```

### build manifest의 속성

| 속성 | 설명 |
|------|------|
| `serverBundles` | 번들 ID를 해당 번들의 `id`와 `file`에 매핑하는 객체 |
| `routeIdToServerBundleId` | 라우트 ID를 해당 서버 번들 ID에 매핑하는 객체 |
| `routes` | 라우트 ID를 라우트 메타데이터에 매핑하는 route manifest. 커스텀 라우팅 계층을 구동하는 데 사용 |

---

## 정리

```
빌드 결과물
├── authenticated/       ← "authenticated" 번들 ID로 만들어진 디렉터리
│   └── index.js         ← _authenticated 하위 라우트들의 request handler
└── unauthenticated/     ← "unauthenticated" 번들 ID
    └── index.js         ← 나머지 라우트들의 request handler
```

- 앱 앞단의 커스텀 라우팅 계층이 `buildManifest.routes`와 `routeIdToServerBundleId`를 보고
  들어온 요청을 어느 번들로 보낼지 결정한다.
