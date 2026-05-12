# Signals: Baggage

> 원문: https://opentelemetry.io/docs/concepts/signals/baggage/

---

## 왜 필요하냐면

요청 시작할 때 `userId` 알고 있음. 이 요청이 여러 서비스 거치면서 다운스트림 서비스들도 이 `userId` 가 필요하다면?

방법 1 (비효율) → 각 서비스가 DB에서 userId 다시 조회
방법 2 (좋음) → 컨텍스트랑 같이 userId 자동으로 전파

Baggage가 방법 2임.

---

## Baggage가 뭔지

**컨텍스트랑 함께 서비스 경계 넘어서 전파되는 key-value 데이터.**

```
프론트엔드                백엔드 API              DB 서비스
┌──────────────┐         ┌──────────────┐        ┌──────────────┐
│ userId: 123  │ ──────► │ userId: 123  │ ─────► │ userId: 123  │
└──────────────┘         └──────────────┘        └──────────────┘
              HTTP 헤더로 자동 전파
```

HTTP 헤더로 이렇게 전달됨:

```
baggage: userId=123,clientId=abc,region=kr
```

---

## 우리 코드에서는

직접 Baggage API 안 씀. 대신 **`business-events.ts` 에서 attributes 로 명시적 전달**:

```typescript
export function recordOrderCreateFailed({ error, checkoutId, memberId }) {
  recordErrorEvent('order.create_failed', error, {
    'commerce.checkout.id': checkoutId,
    'user.id': memberId,       // ← Baggage 대신 Attributes 로 직접 전달
    'error.message': error.message,
  });
}
```

왜 Baggage 안 쓰냐? 우리 프론트엔드는 단일 앱이라 서비스 간 전파가 필요 없음. 이벤트 기록 시점에 필요한 값 명시적으로 넘기는 게 더 안전하고 명확함.

---

## Baggage API 쓰는 방법 (참고용)

```typescript
import { context, propagation } from '@opentelemetry/api';

// Baggage에 값 설정
const bag = propagation.createBaggage({
  'userId': { value: '123' },
});
const newContext = propagation.setBaggage(context.active(), bag);

// Baggage에서 값 읽기
const baggage = propagation.getBaggage(context.active());
const userId = baggage?.getEntry('userId')?.value;
```

---

## Baggage vs Span Attributes 차이

| | Baggage | Span Attributes |
|--|---------|----------------|
| 저장 위치 | 컨텍스트 (별도 저장소) | Span 내부 |
| 전파 | 자동으로 모든 다운스트림 | 전파 안 됨 |
| Span 자동 연결 | **안 됨** | 됨 |
| 대시보드 | 명시적으로 추가해야 보임 | 자동으로 보임 |

중요한 거: Baggage는 Span Attributes에 자동으로 안 들어감.

```typescript
// ❌ Baggage에 넣었다고 Span에 자동으로 안 보임
baggage.set('userId', '123');

// ✓ Span에도 보이려면 명시적으로 추가해야 함
span.setAttribute('user.id', '123');
```

---

## 어디에 쓰면 좋은지

요청 시작 시점에만 알 수 있고, 다운스트림 전체에서 필요한 값들:

```
✓ Account ID, User ID, Product ID  → "어떤 사용자가 느린 DB 호출 경험하나?"
✗ 비밀번호, API 키, 개인정보       → 절대 넣지 말 것
```

---

## 보안 주의

Baggage는 **HTTP 헤더로 평문 전송**됨:

```
baggage: userId=123,sessionToken=SECRET  ← 네트워크에서 노출
```

- 민감 정보 절대 금지
- 외부에서 온 Baggage 무조건 신뢰하지 말 것 (위조 가능)
- 서드파티 서비스로 나갈 때 Baggage 포함되는지 확인

---

## Context Propagation이랑 차이

```
Context Propagation     Baggage
trace_id: abc123        userId: 123
span_id:  def456        clientId: xyz
(기술적 추적 정보)       (비즈니스 데이터)
OTel이 필수로 관리       개발자가 선택적으로 추가
```

택배로 비유:
- Context = 운송장 번호
- Baggage = 박스 안에 넣은 메모

---

## 자주 묻는 것들

**Baggage Span Processor가 뭐야?**

Span 생성될 때마다 현재 Baggage 내용을 자동으로 Span Attributes로 복사해주는 컴포넌트임. 매번 수동으로 `span.setAttribute('userId', baggage.get('userId'))` 안 해도 됨. 근데 민감한 Baggage 항목이 Span에 노출될 위험 있으니 주의.

**"무결성 검사 기능 없다"가 무슨 소리야?**

HTTP 헤더에 있는 Baggage는 중간에 누가 바꿔치기해도 OTel이 모름. `baggage: userId=123` 을 `userId=456` 으로 바꿔도 시스템이 알아채지 못함. 신뢰할 수 없는 외부 소스 Baggage는 검증 없이 쓰면 안 됨.
