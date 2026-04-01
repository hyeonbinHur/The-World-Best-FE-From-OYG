# View Transitions

[View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/ViewTransition)를 활용해서 클라이언트 사이드 내비게이션 중 페이지 전환에 부드러운 애니메이션을 적용할 수 있다.

> 핵심 원리: 내비게이션 업데이트를 `document.startViewTransition()` 으로 감싸면 브라우저가 이전/이후 상태를 스냅샷 찍어서 자동으로 전환 애니메이션을 처리해줌

<br>

## 1. Basic View Transition

### 1.1 Link/NavLink/Form 에 prop 추가

가장 간단한 방법. `viewTransition` prop 하나만 달면 끝.

```tsx
<Link to="/about" viewTransition>
  About
</Link>
```

- 추가 CSS 없이도 기본 **크로스페이드(cross-fade)** 애니메이션 자동 적용
- 내부적으로 `document.startViewTransition()` 호출을 React Router가 대신 처리해줌

<br>

### 1.2 useNavigate 로 프로그래매틱 내비게이션

버튼 클릭 등 코드로 직접 이동할 때는 옵션에 `viewTransition: true` 전달.

```tsx
import { useNavigate } from "react-router";

function NavigationButton() {
  const navigate = useNavigate();

  return (
    <button
      onClick={() =>
        navigate("/about", { viewTransition: true })
      }
    >
      About
    </button>
  );
}
```

> Link에 prop 다는 것과 동일한 크로스페이드 효과. 결국 같은 API를 쓰는 것.

<br>

## 2. Image Gallery 예제

단순 크로스페이드에서 한 발 더 나아가, **특정 요소가 두 페이지 사이를 이동하는 것처럼 보이는** 공유 요소 전환(shared element transition)을 구현하는 예제.

핵심은 `view-transition-name` CSS 속성으로 같은 이름을 리스트 뷰와 상세 뷰 양쪽에 붙여주는 것.

<br>

### 2.1 이미지 갤러리 라우트 생성

```tsx
import { NavLink } from "react-router";

export const images = [
  "https://remix.run/blog-images/headers/the-future-is-now.jpg",
  "https://remix.run/blog-images/headers/waterfall.jpg",
  "https://remix.run/blog-images/headers/webpack.png",
  // ...
];

export default function ImageGalleryRoute() {
  return (
    <div className="image-list">
      <h1>Image List</h1>
      <div>
        {images.map((src, idx) => (
          <NavLink
            key={src}
            to={`/image/${idx}`}
            viewTransition
          >
            <p>Image Number {idx}</p>
            <img className="max-w-full contain-layout" src={src} />
          </NavLink>
        ))}
      </div>
    </div>
  );
}
```

<br>

### 2.2 리스트 뷰 전환 스타일 추가

```css
/* 이미지 그리드 레이아웃 */
.image-list > div {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  column-gap: 10px;
}

.image-list img {
  max-width: 100%;
  contain: layout;
}

.image-list p {
  width: fit-content;
}

/* 핵심: 내비게이션 중일 때만 transition name 부여 */
.image-list a.transitioning img {
  view-transition-name: image-expand;
}

.image-list a.transitioning p {
  view-transition-name: image-title;
}
```

> `a.transitioning` 클래스는 NavLink가 전환 중일 때 자동으로 붙여줌.
> 전환 중인 링크에만 이름을 부여해야 여러 이미지가 동시에 같은 `view-transition-name`을 갖는 충돌을 방지할 수 있음.

<br>

### 2.3 이미지 상세 라우트 생성

```tsx
import { Link } from "react-router";
import { images } from "./home";
import type { Route } from "./+types/image-details";

export default function ImageDetailsRoute({
  params,
}: Route.ComponentProps) {
  return (
    <div className="image-detail">
      <Link to="/" viewTransition>
        Back
      </Link>
      <h1>Image Number {params.id}</h1>
      <img src={images[Number(params.id)]} />
    </div>
  );
}
```

<br>

### 2.4 상세 뷰 전환 스타일 추가

```css
/* 리스트 뷰의 transition name과 동일하게 맞춰야 함 */
.image-detail h1 {
  font-size: 2rem;
  font-weight: 600;
  width: fit-content;
  view-transition-name: image-title;  /* 리스트의 p 와 연결 */
}

.image-detail img {
  max-width: 100%;
  contain: layout;
  view-transition-name: image-expand; /* 리스트의 img 와 연결 */
}
```

> 리스트 뷰와 상세 뷰에서 동일한 `view-transition-name` 값을 사용해야 브라우저가 같은 요소로 인식하고 자연스러운 이동 애니메이션을 만들어줌.

<br>

## 3. Advanced Usage

CSS 클래스 방식 대신 JS에서 직접 전환 상태를 제어하고 싶을 때 두 가지 방법이 있다.

<br>

### 3.1 Render Props 방식

`NavLink`의 children을 함수로 넘기면 `isTransitioning` 값을 받을 수 있다.

```tsx
<NavLink to={`/image/${idx}`} viewTransition>
  {({ isTransitioning }) => (
    <>
      <p
        style={{
          viewTransitionName: isTransitioning ? "image-title" : "none",
        }}
      >
        Image Number {idx}
      </p>
      <img
        src={src}
        style={{
          viewTransitionName: isTransitioning ? "image-expand" : "none",
        }}
      />
    </>
  )}
</NavLink>
```

- 전환 중일 때만 `view-transition-name`을 부여하고, 아닐 때는 `"none"` 처리
- CSS 클래스보다 조건을 더 세밀하게 제어 가능

<br>

### 3.2 useViewTransitionState 훅

컴포넌트를 분리하거나 NavLink 바깥에서 전환 상태를 알고 싶을 때 사용.

```tsx
function NavImage(props: { src: string; idx: number }) {
  const href = `/image/${props.idx}`;
  // 특정 href로 전환 중인지 여부를 boolean으로 반환
  const isTransitioning = useViewTransitionState(href);

  return (
    <Link to={href} viewTransition>
      <p
        style={{
          viewTransitionName: isTransitioning ? "image-title" : "none",
        }}
      >
        Image Number {props.idx}
      </p>
      <img
        src={props.src}
        style={{
          viewTransitionName: isTransitioning ? "image-expand" : "none",
        }}
      />
    </Link>
  );
}
```

- `NavLink` 대신 `Link`를 써도 훅으로 전환 상태를 알 수 있음
- 컴포넌트 분리가 깔끔하게 가능

<br>

## 정리

| 방법 | 언제 쓰나 |
|------|-----------|
| `viewTransition` prop (Link/NavLink/Form) | 기본 크로스페이드로 충분할 때 |
| `useNavigate` + `viewTransition: true` | 코드로 이동할 때 |
| CSS `view-transition-name` | 특정 요소를 공유 요소로 연결할 때 |
| NavLink render props `isTransitioning` | NavLink 내부에서 JS로 전환 상태 제어할 때 |
| `useViewTransitionState` hook | 컴포넌트 분리하거나 Link 바깥에서 상태 필요할 때 |

<br>

> **주의사항**
> - 같은 페이지에 동일한 `view-transition-name`을 가진 요소가 둘 이상 존재하면 전환이 망가짐
> - 그래서 리스트에서는 `a.transitioning` 처럼 전환 중인 요소 하나에만 이름을 부여하는 게 핵심
> - 브라우저 지원: 크로미움 계열은 잘 되지만 파이어폭스/사파리는 아직 부분 지원이므로 필요시 확인
