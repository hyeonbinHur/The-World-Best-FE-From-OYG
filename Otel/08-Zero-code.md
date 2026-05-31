# Zero-Code Instrumentation (무코드 계측)

> 원문: https://opentelemetry.io/docs/concepts/instrumentation/zero-code/

---

## 한 줄 요약

코드 한 줄 안 고치고, 에이전트만 붙이면 자동으로 텔레메트리 수집해주는 방식.

---

## 이게 뭐냐면

앱 소스 코드를 건드리지 않고도 OTel API/SDK를 자동으로 주입해주는 방식임.
보통 **에이전트**를 설치하면 끝.

```
[기존 앱] + [OTel Agent 설치] = 자동으로 Trace/Metric/Log 수집
```

운영팀이 "개발팀 코드 수정 안 기다려도 되나?" → **됨.** 이게 Zero-code의 핵심 가치.

---

## 어떻게 동작하나

언어마다 기법이 다름:

| 언어 | 기법 | 원리 |
|------|------|------|
| **Java** | Bytecode manipulation | `-javaagent` 옵션으로 JVM 에이전트 붙이면, 클래스 로딩 시점에 바이트코드를 끼워넣음. 코드 자체는 안 바뀜 |
| **Python** | Monkey patching | 런타임에 `requests`, `flask` 같은 라이브러리 함수를 OTel 래퍼로 교체. 원본 함수를 감싸는 방식 |
| **JavaScript** | Monkey patching | `require`/`import` 시점에 HTTP, Express 등의 모듈을 가로채서 Span 생성 코드를 주입 |
| **Go** | eBPF injection | 커널 레벨에서 함수 호출을 가로챔. 바이너리 수정 없이 동작 |

쉽게 비유하면:
- **Bytecode manipulation** = 공장에서 제품 조립 중간에 검사 장비를 끼워넣는 것
- **Monkey patching** = 기존 택배 기사(함수)를 OTel 유니폼 입힌 기사로 교체하는 것
- **eBPF** = 도로에 CCTV 달아서 차량(함수 호출)을 감시하는 것

---

## 뭘 자동으로 잡아주나

**라이브러리 레벨의 I/O 작업**을 자동 계측함:

- **HTTP 요청/응답** — 인바운드(서버로 들어오는 요청) + 아웃바운드(다른 서비스 호출)
- **Database 호출** — SQL 쿼리, MongoDB 연산 등
- **Message Queue** — Kafka produce/consume, RabbitMQ 등

예를 들어 Express 서버에 OTel Agent 붙이면:

```
GET /api/orders → 자동으로 Span 생성됨
  ├── http.method: GET
  ├── http.url: /api/orders
  ├── http.status_code: 200
  └── duration: 45ms
```

개발자가 코드에 `tracer.startSpan()` 같은 거 안 써도 자동으로 됨.

---

## 뭘 못 잡나

**비즈니스 로직은 커버 못함.**

```
// 이런 건 Zero-code가 절대 자동으로 안 잡음
function processCheckout(cart) {
  validateCart(cart);        // ← 이 안에서 뭐 했는지?
  calculateDiscount(cart);   // ← 할인 계산이 느린 건지?
  reserveInventory(cart);    // ← 재고 부족이면?
}
```

HTTP 요청이 들어온 것과 DB 쿼리가 나간 건 잡지만,
"장바구니 검증 → 할인 계산 → 재고 확보" 같은 **도메인 흐름**은 모름.

이건 [Code-Based Instrumentation](./09-CodeBased.md)으로 직접 Span 만들어야 함.

---

## 설정 방법

주로 **환경변수**로 설정함. 코드 수정 없이 배포 설정만 바꾸면 됨:

```bash
# 필수: 서비스 이름 (백엔드에서 이 이름으로 식별)
OTEL_SERVICE_NAME=order-service

# Exporter 엔드포인트 (텔레메트리를 어디로 보낼지)
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317

# Context 전파 포맷
OTEL_PROPAGATORS=tracecontext,baggage

# 리소스 속성 (환경, 버전 등 메타데이터)
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,service.version=1.2.0
```

| 설정 항목 | 필수 여부 | 설명 |
|----------|----------|------|
| Service name | **필수** | 백엔드 대시보드에서 이 이름으로 서비스 구분 |
| Exporter | 선택 | 안 쓰면 기본 OTLP stdout |
| Propagator | 선택 | 기본은 W3C TraceContext |
| Resource | 선택 | 환경(prod/dev), 버전 등 부가 정보 |
| Data source | 선택 | 특정 라이브러리 계측 on/off |

---

## 지원 언어

**.NET, Go, Java, JavaScript, PHP, Python**

각 언어별로 설치 방법이 다름. Java는 `-javaagent` JAR, Node.js는 `--require` 플래그 등.

---

## 언제 쓰면 좋나

| 상황 | Zero-code 적합? |
|------|----------------|
| 레거시 서비스에 빠르게 관측 붙이고 싶다 | O |
| 코드 수정 권한이 없다 | O |
| HTTP/DB 수준 계측이면 충분하다 | O |
| 비즈니스 로직(주문, 결제 흐름)까지 추적하고 싶다 | X → Code-based 필요 |
| 커스텀 메트릭(전환율, 실패율)이 필요하다 | X → Code-based 필요 |

---

## 실무 팁

**Zero-code + Code-based를 같이 쓰는 게 일반적임.**

- Zero-code → HTTP, DB, MQ 같은 인프라 레벨 자동 계측
- Code-based → 비즈니스 이벤트, 커스텀 Span 추가

둘이 합치면 이런 그림이 나옴:

```
[Trace]
├── GET /checkout  (Zero-code가 자동 생성)
│   ├── checkout.validate_cart  (Code-based로 직접 생성)
│   ├── checkout.calculate_discount  (Code-based)
│   ├── POST /api/payments  (Zero-code가 자동 생성)
│   │   └── db.query SELECT ...  (Zero-code가 자동 생성)
│   └── checkout.complete  (Code-based)
```
