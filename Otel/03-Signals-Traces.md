# Signals: Traces

> 원문: https://opentelemetry.io/docs/concepts/signals/traces/

---

## 왜 필요하냐면

"결제 버튼 눌렀는데 응답이 3초 걸렸다" → 어디서 3초가 걸린 건지?

Traces 있으면 이렇게 폭포수 다이어그램으로 볼 수 있음:

```
checkout.process  ──────────────────────── 3000ms
  cart.validate   ──── 200ms
  payment.charge         ──────────────── 2600ms  ← 여기 문제
    api.call                ────── 2500ms
```

---

## Span이 뭔지

**작업 하나의 단위. 시작부터 끝까지.**

```json
{
  "name": "payment.charge",
  "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
  "span_id": "5fb397be34d26b51",
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114Z",
  "end_time":   "2022-04-29T18:52:58.687Z",
  "status_code": "STATUS_CODE_ERROR",
  "attributes": {
    "payment.method": "card",
    "http.response.status_code": 500
  }
}
```

---

## Trace가 뭔지

**요청이 통과한 전체 경로. 같은 trace_id 가진 Span들 묶음.**

```
Trace ID: 5b8aa5a2...
│
├── [Root Span] checkout.process   (parent_id 없음)
│   │
│   ├── [Child] cart.validate
│   │
│   └── [Child] payment.charge
│       │
│       └── [Grandchild] api.call
```

---

## 우리 코드에서는

### `trace.ts` — Span 생성/관리 핵심 유틸

```typescript
// Tracer 가져오기
function getTracer() {
  return trace.getTracer('frontend-observability');
}

// 작업을 Span으로 감싸기
export function traceTask(spanName, fn, options) {
  return tracer.startActiveSpan(spanName, (span) => {
    try {
      const result = fn(span, ctx);
      span.end();           // 성공 → 종료
      return result;
    } catch (error) {
      markSpanError(span, error);  // 실패 → 에러 기록
      span.end();
      throw error;
    }
  });
}

// 에러 기록
function markSpanError(span, error) {
  span.recordException(error);
  span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
  span.setAttribute('error.type', error.name);
}
```

실제 사용:

```typescript
await traceTask('checkout.process', async (span, ctx) => {
  span.setAttribute('cart.id', cartId);

  await traceTask('cart.validate', async () => { ... }, { ctx });   // ctx 넘겨서 parent-child 연결
  await traceTask('payment.charge', async () => { ... }, { ctx });
});
```

### `observers.ts` — 자동으로 에러 Span 생성

```typescript
// traceTask 안 쓰고 직접 span 만드는 방식
window.addEventListener('unhandledrejection', (event) => {
  const span = tracer.startSpan('client.unhandled_rejection');
  span.setAttributes({ 'error.type': error.name });
  span.recordException(error);
  span.setStatus({ code: SpanStatusCode.ERROR });
  span.end();
});
```

---

## Span 구성 요소들

### Attributes — 메타데이터

```typescript
span.setAttribute('http.method', 'GET');
span.setAttribute('user.id', userId);
```

Semantic Attributes 쓰면 (`http.method`, `db.system` 등) 대시보드에서 자동 인식됨.

### Span Events — 특정 시점 이벤트

```typescript
// trace.ts 에서 export 해둔 거
export function addSpanEvent(span, eventName, attributes) {
  span.addEvent(eventName, attributes);
}

addSpanEvent(span, 'cart.item_added', { 'product.id': 'SKU-123' });
```

Attribute vs Event 구분:
- 타임스탬프가 중요한 순간 → **Event**
- 시간 상관없이 Span 전체에 적용되는 속성 → **Attribute**

### Span Status

| 상태 | 의미 | 우리 코드 |
|------|------|----------|
| `Unset` (기본) | 오류 없이 완료 | 그냥 `span.end()` |
| `Error` | 에러 발생 | `markSpanError()` |
| `Ok` | 명시적으로 성공 선언 | 거의 안 씀 |

### Span Kind

| Kind | 언제 |
|------|------|
| `Client` | fetch 요청 보낼 때 |
| `Server` | 요청 받는 서버 |
| `Internal` | 내부 처리 (우리 traceTask 대부분 이거) |
| `Producer` | 메시지 큐에 넣을 때 |
| `Consumer` | 메시지 큐에서 꺼낼 때 |

---

## 컴포넌트 흐름

```
앱 시작 시 한 번
┌──────────────────┐
│  TracerProvider  │  ← HyperDX SDK가 처리
└────────┬─────────┘
         │ getTracer()
         ▼
┌──────────────────┐
│     Tracer       │  ← trace.ts 의 getTracer()
└────────┬─────────┘
         │ startActiveSpan()
         ▼
┌──────────────────┐
│      Span        │  ← traceTask(), observers.ts
└────────┬─────────┘
         │ OTLP HTTP
         ▼
┌──────────────────┐
│     HyperDX      │
└──────────────────┘
```

---

## 자주 묻는 것들

**`traceTask` 랑 직접 `startSpan` 차이가 뭐야?**

`traceTask` 는 에러 처리, async/sync 구분, forceFlush 다 알아서 해주는 래퍼임. `observers.ts` 처럼 단순 이벤트 기록은 직접 `startSpan` 써도 됨.

**`flushOnEnd: true` 언제 써?**

페이지 이동/닫히기 직전 중요한 Span에만 씀. 브라우저 닫히면 큐에 있는 데이터 전송 안 될 수 있어서 즉시 `forceFlush` 강제하는 거임. 남용하면 성능 문제 있음.

**Span Links가 뭔데?**

parent-child 관계로 연결 못 하는 비동기 흐름 (메시지 큐, 배치 처리) 에서 두 Trace를 인과적으로 연결하는 거임. 우리 프로젝트에선 현재 안 씀.
