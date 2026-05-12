# Signals: Metrics

> 원문: https://opentelemetry.io/docs/concepts/signals/metrics/

---

## 왜 필요하냐면

Traces는 "이 요청이 왜 실패했나?" 알려줌. 근데 이런 건 모름:

- "지난 1시간 에러율 얼마야?"
- "평균 응답시간 트렌드는?"
- "지금 동시에 몇 명이 결제 중이야?"

이런 **집계된 통계 정보** 가 Metrics임. Traces는 개별 요청, Metrics는 집계된 패턴.

---

## Traces vs Metrics

| | Traces | Metrics |
|--|--------|---------|
| 질문 | 이 요청이 왜 느렸나? | 평균적으로 얼마나 느린가? |
| 단위 | 개별 요청 | 집계된 숫자 |
| 저장 비용 | 높음 | 낮음 |
| 알림(Alert) | 어려움 | 쉬움 |

실제로는 같이 씀. Metrics로 이상 탐지 → Traces로 원인 분석.

---

## 우리 프로젝트에서는

직접 Metrics API 호출하는 코드는 없음. **HyperDX SDK** 랑 vite.config.ts에 설정된 OTel 패키지들이 자동으로 수집함:

```typescript
// vite.config.ts
'@opentelemetry/instrumentation-fetch',           // fetch 요청 메트릭
'@opentelemetry/instrumentation-xml-http-request', // XHR 메트릭
```

이것들이 자동으로 HTTP 요청 수, 에러율, 응답시간 분포 수집해줌.

---

## Metric Instruments 종류

### Counter — 누적 증가만 함

```
비유: 자동차 주행거리계 (한 번 올라가면 안 내려옴)
용도: 총 요청 수, 총 에러 수
```

```typescript
const counter = meter.createCounter('http.requests.total');
counter.add(1, { route: '/checkout' });
```

### Gauge — 현재 값 스냅샷

```
비유: 자동차 연료 게이지 (올라갔다 내려갔다)
용도: 현재 활성 사용자 수, 현재 메모리 사용량
```

### Histogram — 값의 분포 통계

```
용도: 응답시간, 요청 크기 분포
"50ms 이하 요청이 몇 %?" 같은 질문에 답 가능
```

Counter vs Gauge 직관적으로 보면:

```
Counter (총 요청 수):   1 → 5 → 12 → 30 → 50  (절대 안 줄음)
Gauge (현재 접속자):    3 → 10 → 7 → 15 → 4   (올라갔다 내려갔다)
```

---

## Aggregation (집계)

여러 측정값 시간 단위로 합치는 거임:

```
개별 측정값: 30ms, 50ms, 200ms, 45ms, 1200ms, 25ms
              ↓ 1분 단위 집계
집계값: 평균 258ms, P99 1200ms, 총 요청 6
```

OTLP 프로토콜이 이 집계값을 HyperDX로 보냄.

---

## 어떤 도구 골라야 하는지

```
측정값이 절대 안 줄어?
├── Yes → Counter
└── No → 분포가 필요해?
          ├── Yes → Histogram (응답시간, 크기)
          └── No → Gauge (현재 값 스냅샷)
```

---

## 자주 묻는 것들

**Histogram이 왜 필요해? 평균으로 충분하지 않아?**

평균에 속으면 안 됨:

```
9개 응답시간: 10ms × 8개, 9000ms × 1개
평균: 1009ms  ← 이상해 보임
P99: 9000ms   ← 실제 문제
P50: 10ms     ← 대부분은 빠름
```

Histogram으로 "99번째 백분위 응답시간" 을 봐야 진짜 사용자 경험 파악 가능.

**동기(Sync) vs 비동기(Async) Instrument 차이가 뭐야?**

- **동기** → 코드 실행될 때 직접 값 기록 (`counter.add(1)`)
- **비동기** → 수집 시점에 콜백 호출해서 현재 값 가져옴

```typescript
// 비동기: 수집 시점에 OS에 "지금 메모리 얼마야?" 물어봄
meter.createObservableGauge('memory.usage', (obs) => {
  obs.observe(process.memoryUsage().heapUsed);
});
```

**Views가 뭐야?**

수집된 메트릭 출력 방식 커스터마이즈하는 거임. 어떤 메트릭 무시할지, 집계 방식 바꿀지, 어떤 속성 보고할지 설정 가능.
