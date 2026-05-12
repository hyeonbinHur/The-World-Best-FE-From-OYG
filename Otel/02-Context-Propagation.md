# Context Propagation

> 원문: https://opentelemetry.io/docs/concepts/context-propagation/

---

## 왜 필요하냐면

결제 버튼 눌렀을 때 API → 결제 서버 → DB 거친다고 치자. 각 서비스는 독립적으로 실행됨.

"이 세 서비스 작업이 원래 하나의 요청에서 비롯됐다" 는 걸 어떻게 알아? → Context Propagation 이 이걸 해결함.

요청이 서비스 넘어갈 때 "이거 같은 요청임" 이라는 정보를 HTTP 헤더에 실어서 같이 보냄.

---

## Context가 뭔지

서비스들이 서로 연관짓기 위해 필요한 정보 담은 객체임. 핵심은 이 두 개:

- `trace_id` → 전체 요청 식별 ID (모든 Span이 공유)
- `span_id` → 현재 Span의 ID (다음 서비스에서 parent로 씀)

```
프론트엔드                      백엔드 API
┌─────────────────┐            ┌─────────────────┐
│ trace_id: abc   │  요청 전송 │ trace_id: abc   │  ← 같은 trace_id
│ span_id:  111   │ ─────────► │ parent_id: 111  │  ← 프론트 span이 부모
└─────────────────┘            └─────────────────┘
```

---

## Propagation이 뭔지

Context를 HTTP 헤더에 실어 서비스 간에 이동시키는 메커니즘임.

OTel 기본 방식은 **W3C `traceparent` 헤더**:

```
traceparent: 00-a0892f3577b34da6a3ce929d0e0e4736-f03067aa0ba902b7-01
              ^  ^                               ^                ^
             버전  Trace ID (32자)              Parent Span ID  플래그
```

택배로 비유하면:
- Trace ID = 운송장 번호 (물류 전체 추적)
- Span ID = 현재 배송 단계 ID (다음 단계에서 "이전 단계가 부모" 라고 연결)

---

## 우리 코드에서는

`trace.ts` 의 `traceTask` 가 이걸 담당함:

```typescript
export function traceTask(spanName, fn, options) {
  const { ctx } = options ?? {};

  // ctx 있으면 → 그 컨텍스트 아래 자식 Span 생성
  // ctx 없으면 → 현재 활성 컨텍스트 사용
  const parentCtx = ctx ?? otelContext.active();

  return tracer.startActiveSpan(spanName, spanOptions, parentCtx, (span) => {
    const spanCtx = trace.setSpan(parentCtx, span);
    return fn(span, spanCtx);  // fn한테 span이랑 ctx 같이 전달
  });
}
```

실제로 이렇게 씀:

```typescript
// 부모 Span
traceTask('checkout.process', (span, ctx) => {
  // ctx 자식한테 넘기면 parent-child 관계 생김
  traceTask('payment.verify', () => { ... }, { ctx });  // ← 이게 컨텍스트 전파
});
```

---

## Log에서도 컨텍스트 전파됨

`logger.ts` 에서 현재 Span의 traceId 자동으로 로그에 박아넣음:

```typescript
private getTraceId(): string | undefined {
  const span = trace.getActiveSpan();     // 지금 활성 Span 가져오기
  return span?.spanContext().traceId;      // 그 Span의 trace_id 반환
}
```

덕분에 "이 에러 로그가 어느 Trace에서 났나?" 대시보드에서 바로 추적 가능.

---

## Metrics에서도 씀

호출 맥락별로 메트릭 집계 가능:

| 엔드포인트 | 초당 요청 | 평균 응답시간 |
|-----------|----------|-------------|
| `GET /product` (전체) | 370 | 300ms |
| `POST /cart/add` 후 `GET /product` | 330 | 130ms |
| `GET /checkout` 후 `GET /product` | 40 | **1703ms** ← 이상함 |

Context 연결 안 되면 이런 분석 자체가 불가능함.

---

## 보안 주의할 것

- 외부에서 온 traceparent 헤더 무조건 신뢰하지 말 것 (위조 가능)
- 외부 서비스로 내부 trace 정보 내보내지 않도록 주의
- Baggage에 민감 정보 (비밀번호, API 키, 개인정보) 절대 넣지 말 것

---

## 브라우저에서 ctx 명시해야 하는 경우

`setTimeout`, React Query callback, SDK callback 같은 **비동기 경계** 에서 active context 끊길 수 있음.

parent-child 관계가 중요하면 ctx를 명시적으로 넘겨야 함. `trace.ts` 주석에도 이 얘기 있음.

`root: true` 랑 `ctx` 동시에 못 씀. `root` 는 새 Trace 시작, `ctx` 는 기존 Trace 연결이라 서로 반대 의도임.
