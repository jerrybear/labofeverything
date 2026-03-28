# 아이용 영어 문장 학습 서비스 API 입력 출력 스키마 초안

## 목적

이 문서는 바로 개발 가능한 수준으로 MVP API 구조를 정리한 초안이다.

핵심 원칙은 아래와 같다.

- 입력은 짧고 단순하게 받는다.
- 출력은 반드시 구조화한다.
- 한국어 설명은 텍스트 중심으로 다룬다.
- 영어 오디오는 별도 필드로 관리한다.

## 권장 엔드포인트

### `POST /v1/sentence-lessons`

사용자가 영어 문장 1개를 보내면, 설명과 음성 메타데이터를 돌려준다.

## 요청 스키마

```json
{
  "sentence": "I am a boy.",
  "learner_age_band": "5-7",
  "explanation_style": "child",
  "voice_profile": "default_female_us",
  "audio_speed": "normal",
  "include_slow_audio": true,
  "include_word_notes": true,
  "client_request_id": "req_123456"
}
```

## 요청 필드 설명

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| `sentence` | string | 필수 | 사용자가 입력한 영어 문장 |
| `learner_age_band` | string | 선택 | 예: `5-7`, `8-10` |
| `explanation_style` | string | 선택 | 기본값 `child`, 확장 시 `child`, `parent`, `teacher` |
| `voice_profile` | string | 선택 | TTS 보이스 선택 |
| `audio_speed` | string | 선택 | `normal` 또는 `slow` |
| `include_slow_audio` | boolean | 선택 | 느린 음성 추가 생성 여부 |
| `include_word_notes` | boolean | 선택 | 단어별 보충 설명 포함 여부 |
| `client_request_id` | string | 선택 | 클라이언트 추적용 식별자 |

## 서버 내부 처리 단계

1. 입력 정규화
2. 길이 검사
3. 금칙어 및 안전성 검사
4. 난이도 검사
5. 캐시 조회
6. LLM 구조화 출력 생성
7. JSON 검증
8. TTS 생성
9. 응답 조립

## 권장 정규화 규칙

- 앞뒤 공백 제거
- 연속 공백 1개로 축소
- 불필요한 특수문자 제거
- 문장 끝 마침표 보정
- 캐시 키 생성용 소문자 정규화 별도 유지

예:

- 입력: ` i am a boy `
- 표시용 정규화: `I am a boy.`
- 캐시 키용 정규화: `i am a boy.`

## 성공 응답 스키마

```json
{
  "lesson_id": "lesson_01HRX123ABC",
  "status": "ok",
  "input": {
    "original_sentence": "i am a boy",
    "normalized_sentence": "I am a boy.",
    "is_natural": true,
    "natural_rewrite_en": null,
    "difficulty_level": "beginner"
  },
  "explanation": {
    "meaning_ko": "나는 남자아이야.",
    "structure_ko": "I는 나는, am은 ~이다, a boy는 남자아이야 라는 뜻이야.",
    "child_safe_explanation_ko": "I am ... 은 나는 ...이야 라고 말할 때 자주 쓰는 표현이야.",
    "sentence_pattern": "subject + be + noun",
    "word_notes": [
      {
        "word": "I",
        "meaning_ko": "나는"
      },
      {
        "word": "boy",
        "meaning_ko": "남자아이"
      }
    ]
  },
  "audio": {
    "primary": {
      "provider": "elevenlabs",
      "voice_profile": "default_female_us",
      "speed": "normal",
      "format": "mp3",
      "url": "https://cdn.example.com/audio/lesson_01HRX123ABC_normal.mp3",
      "duration_ms": 1450,
      "cache_hit": false
    },
    "slow": {
      "provider": "elevenlabs",
      "voice_profile": "default_female_us",
      "speed": "slow",
      "format": "mp3",
      "url": "https://cdn.example.com/audio/lesson_01HRX123ABC_slow.mp3",
      "duration_ms": 2010,
      "cache_hit": false
    }
  },
  "meta": {
    "age_band": "5-7",
    "tts_text_en": "I am a boy.",
    "created_at": "2026-03-24T10:00:00Z",
    "model_version": "explain-v1",
    "tts_version": "voice-v1"
  }
}
```

## 핵심 응답 필드 설명

| 필드 | 설명 |
| --- | --- |
| `is_natural` | 입력 문장이 자연스러운지 여부 |
| `natural_rewrite_en` | 어색한 경우 추천 자연 표현 |
| `meaning_ko` | 짧은 뜻 설명 |
| `structure_ko` | 아이 눈높이 문장 구조 설명 |
| `child_safe_explanation_ko` | 가벼운 보충 설명 |
| `sentence_pattern` | 내부 분류용 문형 태그 |
| `word_notes` | 단어별 보충 메모 |
| `tts_text_en` | 실제 TTS에 들어간 영어 문장 |

## 실패 응답 스키마

```json
{
  "status": "error",
  "error": {
    "code": "SENTENCE_TOO_COMPLEX",
    "message_ko": "이 문장은 지금 버전에서 설명하기 조금 길거나 어려워요. 더 짧은 문장으로 입력해 주세요.",
    "retryable": true
  }
}
```

## 권장 에러 코드

| 코드 | 의미 |
| --- | --- |
| `EMPTY_SENTENCE` | 입력이 비어 있음 |
| `SENTENCE_TOO_LONG` | 길이 초과 |
| `SENTENCE_TOO_COMPLEX` | 난이도 초과 |
| `UNSUPPORTED_CONTENT` | 현재 지원하지 않는 내용 |
| `SAFETY_BLOCKED` | 안전성 정책 위반 |
| `TTS_GENERATION_FAILED` | TTS 생성 실패 |
| `EXPLANATION_GENERATION_FAILED` | 설명 생성 실패 |

## LLM 출력 스키마

LLM에는 자유 문장을 요청하지 말고 아래 JSON만 반환하게 하는 편이 좋다.

```json
{
  "is_natural": true,
  "natural_rewrite_en": null,
  "tts_text_en": "I am a boy.",
  "meaning_ko": "나는 남자아이야.",
  "structure_ko": "I는 나는, am은 ~이다, a boy는 남자아이야 라는 뜻이야.",
  "child_safe_explanation_ko": "I am ... 은 나는 ...이야 라고 말할 때 자주 쓰는 표현이야.",
  "sentence_pattern": "subject + be + noun",
  "word_notes": [
    {
      "word": "I",
      "meaning_ko": "나는"
    },
    {
      "word": "boy",
      "meaning_ko": "남자아이"
    }
  ]
}
```

## 프롬프트 운영 원칙

- 어려운 문법 용어 금지
- 설명은 1~2문장으로 제한
- 한국어는 쉬운 말투 사용
- 부정확하면 억지 설명보다 `자연스러운 표현 제안` 우선

## 캐시 키 예시

```text
lesson:{normalized_sentence}:{age_band}:{voice_profile}:{audio_speed}:{model_version}:{tts_version}
```

## 저장 대상 권장 필드

- `lesson_id`
- `normalized_sentence`
- `sentence_pattern`
- `meaning_ko`
- `structure_ko`
- `child_safe_explanation_ko`
- `audio_primary_url`
- `audio_slow_url`
- `provider`
- `model_version`
- `tts_version`
- `created_at`

## MVP 범위에서 제외해도 되는 API

- 대화형 음성 채팅 API
- 문장별 발음 채점 API
- 학부모용 고급 리포트 API
- 교사용 학습 진도 API

## 다음 단계 제안

- 이 스키마로 우선 서버 1개 엔드포인트만 구현
- 이후 자주 쓰는 문형에 대해 규칙 기반 응답을 일부 추가
- TTS provider adapter를 두어 교체 가능성 확보
