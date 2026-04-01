# Status Codes

`loader`와 `action`에서 `data` 함수로 HTTP 상태 코드를 직접 설정할 수 있다.

---

## action에서 상태 코드 설정 (400, 201, 200)

폼 데이터를 처리할 때 상황에 따라 다른 상태 코드를 반환한다.

```tsx
// route('/projects/:projectId', './project.tsx')
import type { Route } from "./+types/project";
import { data } from "react-router";
import { fakeDb } from "../db";

export async function action({ request }: Route.ActionArgs) {
  let formData = await request.formData();
  let title = formData.get("title");

  // 400: 잘못된 요청 (유효성 검사 실패)
  if (!title) {
    return data(
      { message: "Invalid title" },
      { status: 400 },
    );
  }

  // 201: 새 리소스 생성 성공
  if (!projectExists(title)) {
    let project = await fakeDb.createProject({ title });
    return data(project, { status: 201 });
  } else {
    // 200: 기본값이라 data() 안 써도 됨
    let project = await fakeDb.updateProject({ title });
    return project;
  }
}
```

| 상황 | 상태 코드 | 설명 |
|------|-----------|------|
| 유효성 검사 실패 | `400` | 잘못된 요청, 에러 메시지 반환 |
| 새 리소스 생성 | `201` | Created, 생성된 데이터 반환 |
| 기존 리소스 수정 | `200` | 기본값이라 `data()` 생략 가능 |

> 폼 에러 렌더링 방법은 **Form Validation** 참고

---

## loader에서 상태 코드 설정 (404)

데이터가 없을 때 `throw`로 ErrorBoundary에 404를 전달한다.

```tsx
// route('/projects/:projectId', './project.tsx')
import type { Route } from "./+types/project";
import { data } from "react-router";
import { fakeDb } from "../db";

export async function loader({ params }: Route.LoaderArgs) {
  let project = await fakeDb.getProject(params.id);

  // 404: 리소스를 찾을 수 없을 때 throw -> ErrorBoundary로 이동
  if (!project) {
    throw data(null, { status: 404 });
  }

  return project;
}
```

> `throw data()`로 던진 에러는 ErrorBoundary에서 처리된다. 자세한 내용은 **Error Boundaries** 참고

---

## 핵심 정리

- `return data(payload, { status })` : 정상 응답에 상태 코드 붙이기
- `throw data(null, { status })` : 에러 상태 코드를 ErrorBoundary로 던지기
- 상태 코드 200은 기본값 → `data()` 없이 그냥 `return` 해도 됨
