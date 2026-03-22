# 나스닥 주식 토큰 거래 문서 안내

## 이 폴더는 무엇을 담고 있나

이 폴더는 `나스닥 주식 토큰 거래`와 `토큰화 증권의 온체인화`를 단계별로 이해할 수 있도록 정리한 연구 문서 모음이다. 처음 읽는 사람부터 전문가 관점으로 보고 싶은 사람까지 순서대로 따라갈 수 있게 구성했다.

## 추천 읽기 순서

### 1. 가장 먼저 보기

- `overview.md`
- 목적: 비전문가도 이해할 수 있는 입문용 요약
- 다루는 내용: 개념, 일반 주식과 차이, 장점, 위험, 현재 상황

### 2. 구조를 이해하고 싶다면

- `expert-system-note.md`
- 목적: 시장 구조와 시스템 구조를 전문가 관점에서 정리
- 다루는 내용: 주문장, 토큰화 플래그, DTC 역할, 청산·결제, 권리 처리

### 3. 온체인화 관점으로 더 깊게 보려면

- `onchainization-deep-dive.md`
- 목적: 어디가 온체인화되고 어디는 아닌지, 완전 온체인 시장과 무엇이 다른지 분석
- 다루는 내용: 온체인/오프체인 레이어, T+0, 담보화, DeFi 연결 가능성

### 4. 근거 출처를 확인하려면

- `sources.md`
- 목적: 어떤 자료를 근거로 문서를 썼는지 빠르게 검증
- 다루는 내용: SEC, Reuters, 법률 해설, 업계 해설의 역할과 신뢰도

### 5. 미래 설계 가설까지 보고 싶다면

- `fully-onchain-securities-market-hypothesis.md`
- 목적: 완전 온체인 증권시장이 어떤 구조를 가져야 하는지 가설 형태로 정리
- 다루는 내용: 발행, 신원, 거래, 원자적 DvP, 권리처리, 규제 내장형 구조

## 문서 간 관계

이 폴더의 문서는 아래 흐름으로 연결된다.

```text
overview
  -> expert-system-note
  -> onchainization-deep-dive
  -> sources
  -> fully-onchain-securities-market-hypothesis
```

- `overview.md`는 전체 지형을 잡아주는 문서다.
- `expert-system-note.md`는 현재 나스닥 구조를 제도권 시장 관점에서 해석한다.
- `onchainization-deep-dive.md`는 그 구조를 온체인화 관점에서 다시 분해한다.
- `sources.md`는 위 문서들의 근거를 정리한다.
- `fully-onchain-securities-market-hypothesis.md`는 현재 구조를 넘어선 미래형 설계 가설이다.

## 빠른 선택 가이드

- `처음 보는 주제`면 `overview.md`
- `전문가 수준 구조 이해`가 목적이면 `expert-system-note.md`
- `온체인화가 어디까지 왔는지`가 궁금하면 `onchainization-deep-dive.md`
- `근거 검증`이 필요하면 `sources.md`
- `미래 설계 상상`이 목적이면 `fully-onchain-securities-market-hypothesis.md`

## 한 줄 요약

이 폴더는 `입문 -> 구조 분석 -> 온체인화 딥다이브 -> 출처 검증 -> 미래 설계 가설` 순으로 읽도록 설계된 나스닥 토큰화 증권 연구 묶음이다.
