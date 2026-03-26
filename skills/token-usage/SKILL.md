---
name: token-usage
description: Claude 프로젝트별 토큰 사용량과 예상 비용을 산출하고 HTML 리포트를 생성한다. "토큰 사용량", "비용 확인", "얼마나 썼어" 등의 요청에 사용.
---

# Token Usage Report Skill

## 개요
`~/.claude/projects/` 아래 JSONL 트랜스크립트를 파싱해 프로젝트별 토큰 사용량과 Sonnet 4.6 기준 예상 비용을 산출하고 HTML 리포트를 생성한다.

## 비용 기준 (claude-sonnet-4-6)
- Input: $3.00 / M tokens
- Cache Write (cache_creation_input_tokens): $3.75 / M tokens
- Cache Read (cache_read_input_tokens): $0.30 / M tokens
- Output: $15.00 / M tokens

## 실행 순서

### 1단계: 트랜스크립트 파싱
`~/.claude/projects/` 아래 모든 폴더의 `*.jsonl` 파일에서 `type == "assistant"` 메시지의 `message.usage` 필드를 집계한다.

```
usage 필드 구조:
{
  "input_tokens": int,
  "cache_creation_input_tokens": int,
  "cache_read_input_tokens": int,
  "output_tokens": int
}
```

### 2단계: 프로젝트 분류

**폴더명 기반 분류** (우선순위 높음):
- `-home-ubuntu-projects-{name}` 형태의 폴더는 `{name}`으로 직접 매핑
- `-home-ubuntu--claude-mem-observer-sessions` → `claude-mem observer`

**cwd 키워드 매칭** (`-home-ubuntu-projects` 폴더 내 혼합 세션):
- 각 세션의 전체 내용에서 프로젝트명 키워드 탐지
- 탐지 키워드 예시:
  - `edu-compass`, `교육나침반` → edu-compass
  - `ai-blogger`, `artcloeai`, `pexels` → ai-blogger
  - `check-in-check` → check-in-check
  - `polymarket`, `polybot` → polybot
  - `paper_compare`, `scalping`, `allweather` → crypto-autotrader
- 매칭 불가 세션은 `(general/mixed)`로 분류

### 3단계: 비용 계산
```python
cost = (
    input_tokens          / 1_000_000 * 3.00 +
    cache_creation_tokens / 1_000_000 * 3.75 +
    cache_read_tokens     / 1_000_000 * 0.30 +
    output_tokens         / 1_000_000 * 15.00
)
```

### 4단계: HTML 리포트 생성
- 어두운 테마 (`#0d1117` 배경)
- 요약 카드: 총 비용 / 총 세션 / 세션당 평균 비용
- 테이블: 프로젝트명, 세션수, 세션당 비용, 총 비용, 비중 바
- 비용 내림차순 정렬
- 저장: `/tmp/token_usage_detailed.html`

### 5단계: 결과 전달
- 주요 수치 텍스트 요약 (상위 3개 프로젝트 + 총계)
- HTML 파일 텔레그램 reply 툴로 전송 (files 파라미터 사용)

## 주의사항
- 비용은 **추정값**임 (실제 플랜/모델 혼용 따라 다를 수 있음)
- cache_read는 저렴하므로 비중이 높아도 실제 비용 기여는 낮음
- claude-mem observer는 자동 실행 세션이 많아 비용이 높게 나오는 경향 있음
- subagents 폴더(`session_id/subagents/*.jsonl`)도 파싱 대상에 포함
