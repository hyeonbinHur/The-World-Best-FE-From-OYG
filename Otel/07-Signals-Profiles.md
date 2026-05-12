# Signals: Profiles

> 원문: https://opentelemetry.io/docs/concepts/signals/profiles/
> **Status: Alpha** (아직 실험적)

---

## 왜 필요하냐면

Traces로 "어떤 Span이 느린지" 알았음. 근데:

> "이 Span 안에서 어떤 함수가 CPU 잡아먹는지" → Traces로는 모름

Profile 있으면:

```
느린 Span 클릭
  → 그 순간 어떤 코드가 실행 중이었나?
     checkout() → validateCart() → calculateTax()  ← 여기서 CPU 80%
```

---

## 4가지 시그널 비교

| 시그널 | 질문 | 상태 |
|--------|------|------|
| Logs | 어떤 이벤트가 발생했나? | Stable |
| Metrics | 시스템 수치는 어때? | Stable |
| Traces | 요청이 어떤 경로 거쳤나? | Stable |
| **Profiles** | **어떤 코드가 리소스 소비하나?** | **Alpha** |

넷 다 있어야 완전한 Observability임.

---

## 우리 프로젝트에서는

**현재 미구현.** Profile은 주로 서버사이드에서 의미 있고, 브라우저 프론트엔드에서 eBPF 기반 프로파일링은 적용 안 됨.

다른 시그널 현황:
- Traces → `trace.ts`, `observers.ts` ✓
- Logs → `logger.ts` ✓
- Metrics → HyperDX SDK 자동 수집 ✓
- Profiles → 미구현

---

## Profile이 뭔지

**실행 중인 앱이 리소스를 어디서 쓰는지 보여주는 스택 트레이스 집합.**

```
On-CPU 프로파일 예시:
main() → handleRequest() → queryDB() → parseSQL()   [CPU 45%]
main() → handleRequest() → renderTemplate()          [CPU 30%]
main() → handleRequest() → validateInput()           [CPU 10%]
```

각 스택 트레이스가 "얼마나 자주 관찰됐나" 로 CPU 사용량 추정함.

---

## Profile 종류

| 종류 | 질문 | 언제 |
|------|------|------|
| **On-CPU** | 어떤 함수가 CPU 쓰나? | 계산이 느린 경우 |
| **Off-CPU** | 어디서 기다리나? | DB 대기, I/O, 락 |
| **Heap** | 어떤 함수가 메모리 보유 중? | 메모리 누수 |
| **Allocations** | 어떤 코드가 메모리 할당하나? | GC 압박 |

On-CPU vs Off-CPU 구분:

```
응답 느린데 CPU 높음 → On-CPU 문제 (계산이 오래 걸림)
응답 느린데 CPU 낮음 → Off-CPU 문제 (DB/IO 기다리는 중)
```

---

## 다른 시그널이랑 연동하면

```
Metrics: "CPU 사용량 90% 급등"
    ↓
Profile: "calculateTax() 함수가 CPU 70% 소비"
    ↓
Traces:  "해당 함수가 느린 Span 안에서 실행 중"
    ↓
Logs:    "그 순간 입력 데이터"
```

이동 패턴:
- Metrics 급등 → Profile로 원인 함수 찾기
- 느린 Span → Profile로 그 시간에 뭐 했는지 확인
- OOM 로그 → Profile로 메모리 누수 코드 경로 찾기

---

## 프로파일링 방법

### 샘플링 기반 (일반적인 방법)

10ms마다 스택 트레이스 기록:

```
t=0ms:  main → handleRequest → queryDB     ← 기록
t=10ms: main → handleRequest → renderHTML  ← 기록
t=20ms: main → handleRequest → queryDB     ← 기록
결과: queryDB 가 2/3 = 67% CPU 사용
```

### eBPF 기반 (Linux 서버)

Linux 커널 수준에서 **코드 변경 없이** 프로파일링:
- 앱 코드 수정 불필요
- 오버헤드 낮음 → 프로덕션에서 항상 켜둘 수 있음

---

## 언제 쓰면 좋은지

```
"에러가 났어요"        → Logs, Traces 먼저 봐
"응답이 느려요"        → Traces 봤는데도 원인 모를 때 Profile
"메모리가 부족해요"    → Metrics 봤는데 어떤 코드인지 모를 때 Profile
"CPU가 높아요"         → Metrics 봤는데 어떤 함수인지 모를 때 Profile
```

---

## 자주 묻는 것들

**스택 트레이스가 뭐야?**

특정 시점에 함수가 어떤 순서로 호출됐는지 목록:

```
main()
  └─ handleRequest()
       └─ queryDB()
            └─ executeSQL()  ← 지금 여기서 실행 중
```

프로파일러가 이걸 주기적으로 캡처해서 "어떤 함수에서 시간 많이 쓰나" 파악함.

**eBPF가 왜 좋은 거야?**

기존 프로파일링은 앱 코드에 계측 심어야 했음 (오버헤드 있음). eBPF는 Linux 커널에서 직접 관찰하니까 앱 건드리지 않아도 됨. 오버헤드 1~5% 수준이라 프로덕션에서 항상 켜둬도 됨.

**Alpha 단계라 쓰면 안 돼?**

OTel 스펙이 Alpha인 거지, HyperDX 같은 도구는 이미 자체 프로파일링 지원함. OTel 표준화 완성되면 더 많은 도구랑 호환될 예정.
