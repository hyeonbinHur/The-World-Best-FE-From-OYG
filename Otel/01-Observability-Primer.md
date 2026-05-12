# Observability Primer

> 원문: https://opentelemetry.io/docs/concepts/observability-primer/

---

## Observability가 뭐냐면

시스템 내부를 몰라도 외부에서 "지금 뭔 일 일어나고 있어?" 라고 물어볼 수 있는 능력임.

실제로 이런 상황에서 빛을 발함:

> 고객이 "결제가 안 돼요" 연락옴
>
> - OTel 없을 때 → 서버 살아있는데 왜 안 되지? 로그 뒤져봐야지... (막막)
> - OTel 있을 때 → 대시보드 열면 `checkout.create_failed` 이벤트 급증 바로 보임. 어떤 에러인지, 어떤 유저인지 즉시 파악

---

## 시그널 3가지 (Traces, Metrics, Logs)

이게 OTel의 핵심임. 시스템이 내뿜는 데이터를 **텔레메트리(Telemetry)** 라고 부르는데, 종류가 3가지.

**Traces** — "이 요청이 어떤 경로로 갔나?"
**Metrics** — "에러율이 지금 얼마나 되나?"
**Logs** — "그 순간 정확히 뭐라고 찍혔나?"

셋 다 있어야 제대로 디버깅 가능.

---

## 우리 프로젝트 구조

```
apps/web/src/shared/observability/
├── opentelemetry/
│   ├── trace.ts           → Span 생성/관리 유틸
│   ├── observers.ts       → 에러/성능 자동 감지
│   └── business-events.ts → 비즈니스 이벤트 기록
└── logger.ts              → 구조화 로그
```

**`observers.ts`** 에서 이런 식으로 자동으로 에러 감지:

```typescript
window.addEventListener("unhandledrejection", (event) => {
  const span = tracer.startSpan("client.unhandled_rejection");
  span.setAttributes({ "error.type": error.name });
  span.recordException(error);
  span.setStatus({ code: SpanStatusCode.ERROR });
  span.end();
  // → HyperDX 대시보드에 바로 뜸
});
```

**`business-events.ts`** 에서는 주문 실패 같은 비즈니스 이벤트 직접 기록:

```typescript
export function recordOrderCreateFailed({ error, checkoutId, memberId }) {
  HyperDX.addAction("order.create_failed", {
    "commerce.checkout.id": checkoutId,
    "user.id": memberId,
    "error.message": error.message,
  });
}
```

---

## Span이랑 Trace 차이

헷갈릴 수 있는데 이렇게 생각하면 됨:

```
사용자 "결제하기" 클릭
│
├── [Trace 시작]  trace_id: "5b8aa5a2..."  ← 요청 전체를 묶는 ID
│   │
│   ├── [Span 1] checkout.create  200ms
│   │   └── [Event] "cart validated" at 50ms
│   │
│   ├── [Span 2] payment.process  1500ms  ← 여기서 느림
│   │   └── SpanStatusCode.ERROR
│   │
│   └── [Log] "payment failed: timeout"
│             traceId="5b8aa5a2..."  ← 이 로그가 어느 Trace에서 났는지 연결됨
```

- **Span** → 작업 하나 (시작~끝)
- **Trace** → 요청 전체 흐름 (여러 Span 묶음)
- **Log** → 특정 시점 메시지 (Trace랑 연결 가능)

---

## 계측(Instrumentation)이 뭔 소린지

코드에 "감지 장치" 심는 거임. 공장 파이프에 온도계 붙이는 거랑 똑같음.

우리 프로젝트는 두 방식 씀:

- **자동 계측** → `observers.ts` 가 window 에러 자동으로 잡음
- **수동 계측** → `business-events.ts` 에서 직접 이벤트 기록

---

## 신뢰성(Reliability)이란

"서비스가 사용자 기대대로 동작하냐?" 이거임.

서버 업타임 100%여도 장바구니에 잘못된 상품 담기면 신뢰할 수 없음. 그래서 `business-events.ts` 에서 `checkout.verify_cart_failed` 같은 이벤트 기록하는 거임.

SLI/SLO는 이 신뢰성을 수치로 관리하는 개념:

- **SLI** → 사용자 관점 지표 ("결제 API 응답시간")
- **SLO** → 그 지표의 목표치 ("99%의 요청이 1초 이내")

---

## "unknown unknowns" 가 뭔지

"내가 모른다는 것조차 모르는 문제"임.

특정 조건에서만 결제 실패하는데 그 조건 자체를 모르는 상황. Observability 있으면 데이터로 패턴 발견 가능.

`observers.ts` 의 `setupGlobalErrorHandlers()` 가 이런 예상치 못한 에러 자동으로 잡아주는 이유임.
