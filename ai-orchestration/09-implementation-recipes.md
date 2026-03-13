# AI 오케스트레이션 구현 레시피

이 문서는 대표적인 AI 오케스트레이션 시나리오별로 "어떤 계층 조합을 기본값으로 잡으면 좋은가"를 빠르게 볼 수 있게 정리한 템플릿 문서다.

## 먼저 한 문장으로

구현 레시피는 정답 아키텍처가 아니라, 문제 유형별로 실패 확률이 낮은 기본 조합을 제안하는 출발점이다.

## 이 문서를 어떻게 보면 좋은가

각 레시피는 아래 질문에 답하도록 구성했다.

- 어떤 문제를 푸는가
- 어떤 패턴을 먼저 쓰는가
- 어떤 도구 조합이 자연스러운가
- 어디에 사람이 개입하는가
- 무엇을 먼저 측정해야 하는가

## 레시피 1: 리서치 자동화

### 추천 기본 조합

- 앱 / 진입점: 가벼운 웹 UI 또는 내부 요청 폼
- 에이전트 계층: `LangGraph`
- 검색 / 데이터 계층: 웹 검색 + 내부 문서 + 벡터DB
- 관측 / 평가 계층: `LangSmith` 또는 `Phoenix`, `DeepEval`

### 먼저 떠올릴 패턴

- planner-executor
- reviewer / critic
- parallel fan-out / fan-in

### 왜 이 조합이 잘 맞는가

리서치 자동화는 조사 -> 분석 -> 검증 -> 문서화 루프가 반복되므로, 상태와 검토 흐름을 명확하게 다룰 수 있는 구성이 유리하다.

### 사람 개입 지점

- 최종 아웃풋 승인
- 인용 검증 샘플 확인
- 고비용 탐색 재실행 승인

### 먼저 볼 지표

- grounding / citation accuracy
- coverage
- cost per completed brief

## 레시피 2: 고객지원 자동화

### 추천 기본 조합

- 앱 계층: `Vercel AI SDK` 또는 상담 채널 연동 계층
- 에이전트 계층: `LangGraph` 또는 `CrewAI`
- 업무 시스템 계층: CRM, 티켓, 주문/결제 API
- 관측 / 평가 계층: `Langfuse` 또는 `Phoenix`, 정책 감사 로깅

### 먼저 떠올릴 패턴

- router
- human-in-the-loop
- reviewer / policy check

### 왜 이 조합이 잘 맞는가

고객지원은 입력 분류, 정책 확인, 실제 액션 실행, 사람 전환이 모두 중요하므로 라우팅과 권한 통제가 핵심이 된다.

### 사람 개입 지점

- 환불, 결제, 민감정보 변경
- 불만 고객 escalations
- 낮은 확신도 응답 검토

### 먼저 볼 지표

- first response time
- no-human resolution rate
- policy compliance rate
- handoff quality

## 레시피 3: 내부 업무 자동화

### 추천 기본 조합

- 통합 / 이벤트 계층: `n8n`
- 에이전트 계층: `CrewAI` 또는 `LangGraph`
- 워크플로 계층: `Temporal` 또는 `Prefect`
- 관측 / 평가 계층: `Phoenix` 또는 `Langfuse`

### 먼저 떠올릴 패턴

- event-driven orchestration
- human-in-the-loop
- planner-executor

### 왜 이 조합이 잘 맞는가

내부 업무 자동화는 메일, 슬랙, ERP, 문서 시스템처럼 여러 시스템을 넘나들고 승인 대기가 많아서 durable execution이 특히 중요하다.

### 사람 개입 지점

- 승인 절차
- 예외 처리
- 고위험 액션 확인

### 먼저 볼 지표

- automation success rate
- cycle time reduction
- human intervention rate
- rework rate

## 레시피 4: 장기 실행형 승인 워크플로

### 추천 기본 조합

- 에이전트 계층: `LangGraph`
- 워크플로 계층: `Temporal`
- 관측 계층: `LangSmith` 또는 `Phoenix`

### 먼저 떠올릴 패턴

- human-in-the-loop
- event-driven orchestration
- reviewer / policy check

### 왜 이 조합이 잘 맞는가

몇 시간 또는 며칠 동안 승인 대기와 외부 신호를 기다려야 할 때는 에이전트보다 workflow durability가 더 큰 차이를 만든다.

### 먼저 볼 지표

- resume success rate
- duplicate action rate
- approval latency

## 아주 빠르게 고르는 규칙

| 상황 | 기본 조합 |
| --- | --- |
| 리서치와 문서 작성 중심 | `LangGraph` + 검색/RAG + `LangSmith`/`DeepEval` |
| 고객지원과 정책 실행 중심 | `LangGraph`/`CrewAI` + CRM/API + tracing/audit |
| 내부 시스템 연결과 승인 중심 | `n8n` + `Temporal`/`Prefect` + 에이전트 계층 |
| 장기 실행과 복구 중심 | 에이전트 계층 + `Temporal` |

## 시작할 때 흔한 실수

- 처음부터 모든 기능을 넣어 과한 멀티에이전트 구조를 만듦
- tracing과 evaluation 없이 먼저 자동화만 연결함
- 사람 승인 단계를 넣고도 대기/재개 방식을 설계하지 않음
- 지표 없이 "잘 되는 것 같다" 수준으로 판단함

## 결론

좋은 구현 레시피는 가장 화려한 조합이 아니라, 문제 구조와 리스크에 맞는 최소 조합을 먼저 고르는 데서 시작한다.

## 다음 문서로 넘어가기

이제 `10-glossary.md`에서 앞선 문서 전반에 나온 핵심 용어를 짧게 다시 확인할 수 있다.

## 3줄 요약

- 구현 레시피는 시나리오별로 어떤 계층 조합을 기본값으로 잡을지 빠르게 정리해주는 문서다.
- 리서치, 고객지원, 내부 업무 자동화는 서로 다른 패턴과 도구 조합을 필요로 한다.
- 초반에는 최소 조합으로 시작하고, 운영 지표를 보면서 계층을 추가하는 편이 안전하다.
