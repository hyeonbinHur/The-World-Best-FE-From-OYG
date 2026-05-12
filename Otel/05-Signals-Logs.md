# Signals: Logs

> 원문: https://opentelemetry.io/docs/concepts/signals/logs/

---

## 왜 필요하냐면

Traces로 어떤 Span에서 에러 났는지 알았는데, 그 순간 정확히 어떤 일이 일어났는지는 Log가 알려줌.

근데 기존 로그는 Traces랑 연결이 안 됨:

```
[ERROR] 2024-08-04 12:45:23 - Failed to connect to database
↑ 이게 어떤 요청 중에 발생한 건지? 알 수 없음
```

OTel 로그는 `trace_id` 랑 `span_id` 포함해서 연결됨:

```json
{
  "level": "ERROR",
  "message": "Failed to connect to database",
  "traceId": "5b8aa5a2d2c872e8321cf37308d69df2",
  "spanId":  "5fb397be34d26b51"
}
```

---

## 우리 코드에서는

`logger.ts` 가 OTel 연동 구조화 로거 구현함.

### 핵심: Trace-Log 자동 연결

```typescript
private getTraceId(): string | undefined {
  const span = trace.getActiveSpan();    // 현재 활성 Span 가져오기
  return span?.spanContext().traceId;     // traceId 반환
}

private createLogEntry(level, message, context) {
  return {
    timestamp: new Date().toISOString(),
    level,
    message,
    context: {
      ...context,
      traceId: context?.traceId || this.getTraceId(),  // 자동 주입
    },
  };
}
```

`traceTask` 안에서 `logger.error('결제 실패')` 호출하면 자동으로 현재 Trace의 traceId 포함됨.

### 도메인별 로거

```typescript
// logger.ts
export const cartLogger     = createDomainLogger('cart');
export const checkoutLogger = createDomainLogger('checkout');
export const orderLogger    = createDomainLogger('order');

// 쓸 때
checkoutLogger.error('결제 실패', {
  action: 'payment.charge',
  error: error.message,
});
// → { domain: 'checkout', action: 'payment.charge', error: '...', traceId: '...' }
```

### 성능 측정

```typescript
const result = await logger.time('fetchProducts', async () => {
  return await api.getProducts();
});
// 완료: { action: 'fetchProducts', duration: 234, status: 'ok' }
// 실패: { action: 'fetchProducts', duration: 234, status: 'error', error: '...' }
```

---

## 구조화 로그가 왜 중요한지

**비구조화 (나쁜 예)**

```
[ERROR] 2024-08-04 12:45:23 - Payment failed for user john: card declined, amount 50000
```

"카드 거절 건수" 집계하려면 정규식 파싱 해야 함. 불편함.

**구조화 (좋은 예)**

```json
{
  "level": "ERROR",
  "message": "Payment failed",
  "context": {
    "domain": "checkout",
    "userId": "john",
    "reason": "card_declined",
    "amount": 50000,
    "traceId": "5b8aa5a2..."
  }
}
```

대시보드에서 `reason=card_declined` 바로 필터링 + 해당 Trace 이동 가능.

---

## 로그 형식 3가지

| 형식 | 특징 | 어디서 쓰나 |
|------|------|-----------|
| **구조화** (JSON) | 안정적 스키마, 파싱 쉬움 | 프로덕션 |
| **비구조화** (plain text) | 사람이 읽기 쉬움 | 개발 환경 |
| **준구조화** (key=value) | 중간 단계 | 레거시 |

우리 `logger.ts` 는 dev에서 `console.groupCollapsed` 로 보기 좋게, prod에서 JSON으로 전송함.

---

## Log Record 필드

| 필드 | 우리 코드 |
|------|----------|
| `timestamp` | `new Date().toISOString()` |
| `level` | `LogLevel` 타입 |
| `message` | 첫 번째 인자 |
| `traceId` | `getTraceId()` 자동 주입 |
| `environment` | `import.meta.env.MODE` |
| `version` | `VITE_APP_VERSION` |

---

## Log Bridge가 뭔지

기존 로거 (`winston`, `bunyan`, `console.log`) 를 OTel에 연결하는 어댑터임.

앱 개발자가 직접 쓸 일은 없음. 기존 로거 코드 그대로 두고 OTel 형식으로도 내보내게 해주는 중간 레이어.

우리 `logger.ts` 는 Bridge 없이 `trace.getActiveSpan()` 직접 호출해서 연결하는 커스텀 구현임.

---

## 자주 묻는 것들

**JSON 로그가 무조건 구조화 로그는 아니야?**

맞음. 서로 다른 곳에서 다른 필드 이름 쓰면 구조화가 아님:

```json
// A 서비스: { "userId": "123" }
// B 서비스: { "user_id": "123" }  ← 같은 의미인데 다른 키
```

핵심은 **안정적인 스키마** (항상 같은 필드 이름과 타입).

**`logger.time()` 이랑 `traceTask()` 차이가 뭐야?**

`logger.time()` 은 실행 시간을 로그로 남기고 DataDog RUM에도 보냄. Span은 안 만들어짐. `traceTask()` 는 Span 만들어서 HyperDX 트레이스 뷰에 폭포수 다이어그램으로 보임. 둘 같이 쓰면 더 완전한 그림 나옴.

**배치 설정 (`batchSize: 10`, `flushInterval: 5000`) 이 왜 있어?**

로그 하나씩 즉시 전송하면 네트워크 요청 너무 많아짐. 10개 묶거나 5초마다 한 번에 보내서 오버헤드 줄임. 에러 로그는 샘플링 없이 항상 전송됨.
