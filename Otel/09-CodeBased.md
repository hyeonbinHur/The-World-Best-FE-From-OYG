# Code-Based Instrumentation (코드 기반 계측)

> 원문: https://opentelemetry.io/docs/concepts/instrumentation/code-based/

---

## 한 줄 요약

OTel API/SDK를 코드에 직접 임포트해서, 내가 원하는 곳에 Span/Metric/Log를 만드는 방식.

---

## Zero-code랑 뭐가 다른지

| | Zero-code | Code-based |
|--|----------|-----------|
| 코드 수정 | 안 함 | 직접 API 호출 코드 작성 |
| 잡아주는 것 | HTTP, DB, MQ 등 라이브러리 호출 | **뭐든지** (비즈니스 로직 포함) |
| 세밀함 | 자동이라 뭉뚱그려짐 | 내가 원하는 단위로 정밀 계측 |
| 설정 | 환경변수 | 코드 + 환경변수 |

Zero-code는 "자동 CCTV", Code-based는 "내가 직접 카메라 위치 잡는 것".

---

## 의존성 규칙 (중요)

| 내가 만드는 것 | 의존해야 할 것 | 이유 |
|--------------|--------------|------|
| **라이브러리** (npm 패키지 등) | **API만** | 내 라이브러리 쓰는 사람이 SDK 안 붙여도 에러 안 나게. API만 있으면 no-op(아무것도 안 함)으로 동작 |
| **서비스** (실제 돌아가는 앱) | **API + SDK** | 실제로 텔레메트리를 만들어서 내보내야 하니까 |

쉽게 말하면:
- **API** = 인터페이스 (Tracer, Meter 같은 추상화)
- **SDK** = 구현체 (실제로 Span 만들고, Export하는 로직)

라이브러리가 SDK까지 의존하면 → 사용자 앱이 SDK 버전 충돌 등으로 터질 수 있음.
그래서 라이브러리는 API만 쓰고, SDK는 최종 앱이 선택하게 하는 구조임.

---

## 설정 단계

### Step 1: Provider 초기화 (앱 시작 시 1번)

```
TracerProvider  → Tracer 찍어내는 공장
MeterProvider   → Meter 찍어내는 공장
LoggerProvider  → Logger 찍어내는 공장
```

각 Provider를 만들 때 두 가지를 같이 세팅:
- **Resource** — "이 텔레메트리는 어떤 서비스에서 나온 건지" 메타정보 (서비스명, 버전, 환경)
- **Exporter** — "텔레메트리를 어디로 보낼지" (Collector, Jaeger, stdout 등)

### Step 2: Tracer/Meter 인스턴스 획득

```javascript
const tracer = provider.getTracer("com.example.order-service", "semver:1.2.0");
const meter  = provider.getMeter("com.example.order-service", "semver:1.2.0");
```

이름을 넣는 이유:
- 이 이름이 만들어지는 모든 Span/Metric의 **네임스페이스** 역할
- 나중에 "어떤 라이브러리/서비스가 이 텔레메트리를 만들었지?" 추적 가능
- 버전은 현재 릴리스 버전과 맞추는 게 관례 (`semver:X.Y.Z`)

### Step 3: 텔레메트리 데이터 생성

```javascript
// Trace — Span 만들기
const span = tracer.startSpan("checkout.validate_cart");
span.setAttribute("cart.item_count", cart.items.length);
// ... 비즈니스 로직 ...
span.end();

// Metric — 카운터 찍기
const counter = meter.createCounter("checkout.attempts");
counter.add(1, { "payment.method": "card" });
```

의존하는 외부 라이브러리에 대해선 **Instrumentation Library**가 있는지 먼저 확인.
있으면 그거 쓰면 됨. [OTel Registry](https://opentelemetry.io/ecosystem/registry/)에서 검색 가능.

### Step 4: 데이터 내보내기

두 가지 방식:

| 방식 | 어떻게 | 언제 쓰나 |
|------|-------|----------|
| **In-process Export** | 앱 안에서 Exporter가 직접 Jaeger/Prometheus 포맷으로 변환해서 전송 | 서비스 적고 구성 단순할 때 |
| **OTLP → Collector** | OTLP 프로토콜로 Collector에 보내고, Collector가 백엔드로 중계 | **권장.** 백엔드 바꿔도 앱 코드 안 건드림 |

```
[방식 1] 앱 → Jaeger Exporter → Jaeger 직접 전송
[방식 2] 앱 → OTLP Exporter → Collector → Jaeger/Datadog/등등
```

방식 2가 권장인 이유:
- 백엔드를 Jaeger에서 Datadog으로 바꿀 때 **앱 재배포 없이 Collector 설정만 변경**
- Collector에서 필터링, 배치 처리, 샘플링 등 추가 가능
- 여러 백엔드에 동시 전송(fan-out) 가능

---

## SDK 설정

서비스(실제 돌아가는 앱)에서만 필요. 설정 방법:

- 프로그래밍 방식 (코드에서 직접 Provider 생성)
- 설정 파일 (YAML 등)
- 환경변수

설정 항목:
- Export 대상 (OTLP, Jaeger, Prometheus 등)
- 샘플링 전략 (전부 수집? 10%만?)
- 리소스 속성 (서비스명, 환경, 버전)
- 언어별 튜닝 옵션

---

## 실전 패턴: Zero-code + Code-based 조합

실무에서는 둘을 **같이** 쓰는 게 일반적:

```
[Zero-code가 알아서 잡는 것]
├── HTTP 요청 인바운드/아웃바운드 Span
├── DB 쿼리 Span
└── 메시지 큐 produce/consume Span

[Code-based로 직접 추가하는 것]
├── checkout.validate_cart Span
├── checkout.calculate_discount Span
├── order.create_failed Event
└── checkout.conversion_rate Metric
```

이렇게 하면 인프라 레벨부터 비즈니스 레벨까지 전부 커버됨.
