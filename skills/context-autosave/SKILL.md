---
name: context-autosave
description: |
  컨텍스트 사용량 임계치 도달 시 자동 저장 시스템 설치.
  Stop 훅에서 JSONL을 파싱해 정확한 % 계산 → 설정한 임계치 도달 시 autosave 실행.
  Use when: "컨텍츠 자동 저장 설정", "context auto-save", "75% 저장", "컴팩션 전 자동 저장"
user_invocable: true
allowed-tools:
  - Read
  - Write
  - Bash
---

# Context Auto-Save Setup

## 목적

Claude Code의 컨텍스트가 일정 % 이상 찼을 때 자동으로 저장 스크립트를 실행하는 시스템.
매 응답 후 Stop 훅에서 세션 JSONL(대화 로그)을 파싱해 현재 토큰 사용량 %를 계산하고,
설정한 임계치 도달 시 원하는 저장 스크립트를 실행한다.

컴팩션(자동 컨텍스트 압축) 전에 버퍼 구간을 만들어 중요 정보 유실을 방지하는 게 핵심.

## 동작 원리

```
매 응답 후 (Stop 훅 실행)
  → ~/.claude/projects/.../session.jsonl 파싱
  → 현재 input_tokens + cache 토큰 합산 → % 계산
  → 75% 도달 시: 사용자 정의 autosave 스크립트 실행
  → 90% 도달 시: Claude Code 자동 컴팩션 (별도 PreCompact 훅)
```

> 💡 "transcript_path"는 Alpha Vantage 같은 외부 트랜스크립트가 아님.
> Claude Code가 대화를 기록하는 세션 JSONL 파일 경로 (`~/.claude/projects/.../session-id.jsonl`).
> context-bar.sh (상태바)도 이 파일을 파싱해서 "31% of 200k" 수치를 계산함.

---

## 설치 가이드 (신규 사용자)

### 전제 조건

| 항목 | 필수 여부 | 설명 |
|------|---------|------|
| Claude Code | ✅ 필수 | settings.json과 Stop 훅 지원 버전 |
| `jq` | ✅ 필수 | JSONL 파싱용. `sudo apt install jq` 또는 `brew install jq` |
| 저장 스크립트 | 선택 | 75% 도달 시 실행할 스크립트. 없으면 단계 1에서 기본 스크립트 생성 |
| 알림 스크립트 | 선택 | 텔레그램 등 알림용. 없으면 skip |

### 단계 1: 저장 스크립트 준비

**A) 직접 만든 저장 스크립트가 있으면** — 해당 경로를 메모해두고 단계 2로.

**B) 없으면** — 아래 기본 스크립트 생성 (세션 정보를 텍스트 파일에 append):

```bash
cat > ~/.claude/scripts/simple-autosave.sh << 'EOF'
#!/bin/bash
# 기본 autosave: 현재 날짜/시간을 로그 파일에 기록
# 여기에 원하는 저장 로직을 추가하세요 (git commit, 파일 저장 등)
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Context auto-save triggered" >> ~/.claude/autosave.log
EOF
chmod +x ~/.claude/scripts/simple-autosave.sh
mkdir -p ~/.claude/scripts
echo "기본 저장 스크립트 생성됨: ~/.claude/scripts/simple-autosave.sh"
```

원하는 저장 로직 예시:
- `git -C /your/project commit -am "auto: context checkpoint"` — git 커밋
- `cp ~/important-file ~/backups/important-file.$(date +%s)` — 파일 백업
- `python3 ~/scripts/save-context.py` — 커스텀 Python 스크립트

### 단계 2: context-threshold-save.sh 생성

아래에서 `AUTOSAVE_SCRIPT` 경로를 단계 1에서 준비한 경로로 교체 후 실행:

```bash
mkdir -p ~/.claude/scripts
cat > ~/.claude/scripts/context-threshold-save.sh << 'SCRIPT'
#!/bin/bash
# Stop hook: 컨텍스트 % 체크 → 임계치 도달 시 autosave 실행
# 세션 내 중복 방지: flag file 패턴 사용

SAVE_THRESHOLD=75    # 자동 저장 trigger % (여기서 조정)
AUTOSAVE_SCRIPT="~/.claude/scripts/simple-autosave.sh"  # ← 단계 1 경로로 교체
NOTIFY_SCRIPT=""     # 알림 스크립트 경로 (없으면 빈 문자열)

input=$(cat)
transcript_path=$(echo "$input" | jq -r '.transcript_path // empty')
max_context=$(echo "$input" | jq -r '.context_window.context_window_size // 200000')

if [[ -z "$transcript_path" || ! -f "$transcript_path" ]]; then
    exit 0
fi

# 현재 컨텍스트 % 계산 (Claude Code 상태바와 동일 방식)
context_length=$(jq -s '
    map(select(.message.usage and .isSidechain != true and .isApiErrorMessage != true)) |
    last |
    if . then
        (.message.usage.input_tokens // 0) +
        (.message.usage.cache_read_input_tokens // 0) +
        (.message.usage.cache_creation_input_tokens // 0)
    else 0 end
' < "$transcript_path" 2>/dev/null)

if [[ -z "$context_length" || "$context_length" -eq 0 ]]; then
    exit 0
fi

pct=$((context_length * 100 / max_context))

# 세션 scoped flag (transcript 경로 해시 기반, 중복 방지)
session_hash=$(echo "$transcript_path" | md5sum | cut -c1-8)
flag_file="/tmp/claude-ctx-${SAVE_THRESHOLD}-triggered-${session_hash}"

# 65% 미만: 플래그 리셋
if [[ $pct -lt 65 ]]; then
    rm -f "$flag_file"
    exit 0
fi

# SAVE_THRESHOLD 이상, 아직 trigger 안 됨
if [[ $pct -ge $SAVE_THRESHOLD && ! -f "$flag_file" ]]; then
    touch "$flag_file"
    
    AUTOSAVE_EXPANDED="${AUTOSAVE_SCRIPT/\~/$HOME}"
    [[ -x "$AUTOSAVE_EXPANDED" ]] && bash "$AUTOSAVE_EXPANDED" 2>/dev/null &
    
    if [[ -n "$NOTIFY_SCRIPT" && -f "${NOTIFY_SCRIPT/\~/$HOME}" ]]; then
        echo "{\"message\": \"💾 컨텍스트 ${pct}% — 자동 저장 실행\"}" | \
            python3 "${NOTIFY_SCRIPT/\~/$HOME}" 2>/dev/null &
    else
        echo "💾 Context ${pct}% — auto-save triggered" >> ~/.claude/autosave.log
    fi
fi

exit 0
SCRIPT
chmod +x ~/.claude/scripts/context-threshold-save.sh
echo "생성 완료: ~/.claude/scripts/context-threshold-save.sh"
```

### 단계 3: settings.json 업데이트

```bash
python3 << 'EOF'
import json, os

settings_path = os.path.expanduser("~/.claude/settings.json")

# settings.json이 없으면 빈 구조로 생성
if not os.path.exists(settings_path):
    s = {}
else:
    with open(settings_path) as f:
        s = json.load(f)

# CLAUDE_AUTOCOMPACT_PCT_OVERRIDE → 90 (컴팩션을 늦춤)
s.setdefault("env", {})["CLAUDE_AUTOCOMPACT_PCT_OVERRIDE"] = "90"

# Stop 훅에 context-threshold-save.sh 추가 (중복 방지)
new_hook = {
    "type": "command",
    "command": os.path.expanduser("~/.claude/scripts/context-threshold-save.sh"),
    "timeout": 15,
    "async": True
}
stop_hooks = s.setdefault("hooks", {}).setdefault("Stop", [])
existing = [h.get("command", "") for h in stop_hooks if isinstance(h, dict)]
if new_hook["command"] not in existing:
    stop_hooks.insert(0, new_hook)
    print(f"Stop 훅 추가됨: {new_hook['command']}")
else:
    print("Stop 훅 이미 등록됨 (skip)")

os.makedirs(os.path.dirname(settings_path), exist_ok=True)
with open(settings_path, "w") as f:
    json.dump(s, f, indent=2, ensure_ascii=False)

print(f"AUTOCOMPACT_PCT: {s['env']['CLAUDE_AUTOCOMPACT_PCT_OVERRIDE']}")
print("settings.json 업데이트 완료")
EOF
```

### 단계 4: 검증

```bash
# 스크립트 존재 + 실행 권한 확인
ls -la ~/.claude/scripts/context-threshold-save.sh

# settings.json 확인
python3 -c "
import json, os
with open(os.path.expanduser('~/.claude/settings.json')) as f:
    s = json.load(f)
pct = s.get('env',{}).get('CLAUDE_AUTOCOMPACT_PCT_OVERRIDE','미설정')
stop = [h.get('command','?')[:60] for h in s.get('hooks',{}).get('Stop',[]) if isinstance(h,dict)]
print(f'AUTOCOMPACT_PCT: {pct}')
print(f'Stop 훅: {stop}')
"

# jq 설치 확인
jq --version
```

모든 항목이 에러 없이 출력되면 설치 완료.

---

## 파라미터 조정

`~/.claude/scripts/context-threshold-save.sh` 상단의 변수를 직접 수정:

```bash
SAVE_THRESHOLD=75    # 저장 trigger % 변경
AUTOSAVE_SCRIPT="~/your/save-script.sh"  # 저장 스크립트 교체
NOTIFY_SCRIPT="~/your/notify-script.py"  # 알림 추가 (선택)
```

## 성공 기준

- `~/.claude/scripts/context-threshold-save.sh` 존재 + 실행 권한
- `settings.json`의 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` = `"90"`
- Stop 훅 목록에 스크립트 경로 등록됨
- 다음 Claude Code 세션에서 컨텍스트 75% 도달 시 autosave 실행됨

## 주의사항

- **jq 필수**: JSONL 파싱에 `jq` 명령어 사용. 없으면 아무것도 작동 안 함.
- **Claude Code 재시작 필요**: settings.json 변경은 다음 세션부터 적용.
- **중복 방지**: `/tmp/claude-ctx-75-triggered-*` flag 파일로 세션당 1회만 실행. 시스템 재시작 시 자동 초기화.
- **비동기 실행**: autosave는 백그라운드 실행(`&`) — 응답 지연 없음.
- **SAVE_THRESHOLD < AUTOCOMPACT_PCT**: 저장 임계치는 반드시 컴팩션 임계치보다 낮아야 함 (예: 75 < 90).
