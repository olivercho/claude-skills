# claude-skills

A collection of Claude Code skills by Oliver Cho.

A collection of Claude Code skills — install once, use forever.

---

## Installation

Add to `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "olivercho": {
    "source": { "source": "github", "repo": "olivercho/claude-skills" }
  }
}
```

Then open `/plugins` in Claude Code and search for any skill below.

---

## Skills

### claude-telegram-hooks

Automatically forwards your Claude Code terminal conversations to Telegram — both your messages and Claude's responses.

**Usage:** Tell Claude *"set up Telegram forwarding"* or *"텔레그램 연동 설정해줘"*

### token-usage

Parses Claude Code transcripts to calculate per-project token usage and estimated cost. Generates a dark-themed HTML report and a text summary.

![token-usage report screenshot](assets/token-usage-preview.png)

**Usage:** Tell Claude *"show token usage"*, *"비용 확인해줘"*, or *"얼마나 썼어"*

### hotspot

Analyzes git history to identify the riskiest files, bus factor, team momentum, and firefighting frequency — before reading a single line of code.

**Usage:** Tell Claude *"hotspot 분석해줘"*, *"위험한 파일 찾아줘"*, or *"run hotspot"*

---

## 설치 방법

`~/.claude/settings.json`에 추가:

```json
"extraKnownMarketplaces": {
  "olivercho": {
    "source": { "source": "github", "repo": "olivercho/claude-skills" }
  }
}
```

Claude Code에서 `/plugins` → 원하는 스킬 검색 후 설치.

---

## 스킬 목록

### claude-telegram-hooks

Claude Code 터미널 대화를 텔레그램으로 자동 전달해주는 스킬. 내 메시지와 클로드 답변 모두 실시간으로 받아볼 수 있어요.

**사용법:** 클로드에게 *"텔레그램 연동 설정해줘"* 라고 하면 됩니다.

### token-usage

`~/.claude/projects/` 아래 JSONL 트랜스크립트를 파싱해 프로젝트별 토큰 사용량과 Sonnet 4.6 기준 예상 비용을 산출합니다. 어두운 테마의 HTML 리포트와 텍스트 요약을 생성해 줍니다.

![token-usage report screenshot](assets/token-usage-preview.png)

**사용법:** *"토큰 사용량 확인해줘"*, *"비용 얼마나 썼어"* 라고 하면 됩니다.

### hotspot

git 히스토리만으로 위험 파일·버스 팩터·팀 모멘텀·핫픽스 빈도를 진단합니다. 코드 한 줄 읽기 전에 실행하면 어디서부터 봐야 할지 파악할 수 있어요.

**사용법:** *"hotspot 분석해줘"*, *"위험한 파일 찾아줘"* 라고 하면 됩니다.
