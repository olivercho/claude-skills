---
name: github-watch
description: |
  GitHub 리포지토리 변경사항을 주기적으로 감시하고 Telegram으로 알림을 보내는 워처를 설치/관리한다.
  Use when: "GitHub 리포 워치해줘", "watch this repo", "리포 업데이트 알림", "/github-watch", "github watch"
user_invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
---

# GitHub Repo Watcher

GitHub 리포지토리의 새 커밋을 감지하면 Telegram(또는 다른 채널)으로 알림을 보내는
watcher 스크립트 + 스케줄러를 설치하고 관리하는 스킬.

**지원 OS**: Linux ✅ / macOS ✅ / Windows ✅

---

## 서브커맨드

사용자 요청에서 의도를 파악해 아래 중 하나를 실행:

| 요청 예시 | 서브커맨드 |
|-----------|------------|
| `/github-watch add owner/repo` | **add** |
| `/github-watch list` | **list** |
| `/github-watch remove owner/repo` | **remove** |
| `/github-watch test` | **test** |
| `/github-watch setup` (처음 설치) | **setup** |

서브커맨드가 불명확하면 현재 watch 목록을 보여주고 다음 행동을 물어본다.

---

## OS 감지 (모든 서브커맨드 실행 전)

```bash
python3 -c "import platform; print(platform.system())"
# → "Linux" | "Darwin" | "Windows"
```

Windows이면 `python3` 대신 `python` 또는 `py` 시도:
```batch
python --version 2>/dev/null || py --version
```

이하 절차에서 `{PYTHON}` = 감지된 Python 명령어, `{OS}` = 감지된 OS.

---

## 설정값 (환경마다 다름)

스킬 실행 전 아래 값을 확인한다. 없으면 사용자에게 묻는다.

```
SCRIPT_DIR    : watcher 스크립트 저장 위치
                Linux/Mac 기본: ~/scripts/
                Windows 기본:   %USERPROFILE%\scripts\

STATE_FILE    : SHA 상태 저장 파일
                Linux/Mac: ~/scripts/github_watch_state.json
                Windows:   %USERPROFILE%\scripts\github_watch_state.json

LOG_FILE      : 실행 로그 파일
                Linux/Mac: ~/logs/github_watcher.log
                Windows:   %USERPROFILE%\logs\github_watcher.log

SCHEDULE      : 실행 주기
                Linux/Mac: cron 표현식 (기본 "*/5 * * * *" = 5분마다, Events API + ETag 모드)
                Windows:   Task Scheduler 시간 (기본 every 5 min)

NOTIFIER      : "telegram" | "slack" | "stdout" (기본: telegram)
```

**Telegram 설정 (NOTIFIER=telegram):**
- `TELEGRAM_BOT_TOKEN`: 봇 토큰
- `TELEGRAM_CHAT_ID`: 메시지 받을 채팅 ID

토큰은 `~/.claude/channels/telegram/.env`에서 읽거나 사용자에게 직접 물어본다.

---

## 서브커맨드별 절차

### setup (최초 1회)

1. `SCRIPT_DIR` 디렉토리 확인/생성:
   - Linux/Mac: `mkdir -p {SCRIPT_DIR}`
   - Windows: `if not exist "{SCRIPT_DIR}" mkdir "{SCRIPT_DIR}"`
2. `LOG_FILE` 부모 디렉토리 확인/생성 (동일 패턴)
3. **watcher 스크립트 생성**: `{SCRIPT_DIR}/github_repo_watcher.py` — 아래 [스크립트 템플릿] 참고
4. Linux/Mac만: `chmod +x {SCRIPT_DIR}/github_repo_watcher.py`
5. Telegram credentials 확인
6. **테스트 실행**: `{PYTHON} {SCRIPT_DIR}/github_repo_watcher.py`
   - STATE_FILE 없으면 정상 오류 메시지가 나와야 함 (크래시 아님)
7. 스케줄 등록은 add 서브커맨드에서 처리

### add `<owner/repo>`

1. 입력 형식 검증: `owner/repo` 패턴인지 확인
2. GitHub API로 최신 커밋 SHA fetch:
   ```bash
   # Linux/Mac
   curl -s "https://api.github.com/repos/{owner}/{repo}/commits?per_page=1" -H "User-Agent: github-watcher/1.0"
   # 또는 Python urllib로 (Windows 포함 공통)
   python3 -c "
   import json, urllib.request
   req = urllib.request.Request('https://api.github.com/repos/{owner}/{repo}/commits?per_page=1', headers={'User-Agent':'github-watcher/1.0'})
   data = json.loads(urllib.request.urlopen(req).read())
   print(data[0]['sha'][:7] if data else 'ERROR')
   "
   ```
3. 응답에서 `.sha` 추출 (404면 리포 없음 오류)
4. `STATE_FILE` 읽기 (없으면 빈 `{}`)
5. 리포 키로 엔트리 추가:
   ```json
   {
     "owner/repo": {
       "name": "표시명 (선택, 없으면 repo name 사용)",
       "last_sha": "<현재 SHA>",
       "last_checked": "<ISO timestamp>",
       "last_notified_sha": "<현재 SHA>"
     }
   }
   ```
6. `STATE_FILE` 저장
7. **스케줄 등록** (아직 없으면):

   **Linux/Mac (crontab):**
   ```bash
   # 중복 방지
   crontab -l 2>/dev/null | grep -q github_repo_watcher && echo "already registered" || \
   (crontab -l 2>/dev/null; echo "*/5 * * * * {PYTHON} {SCRIPT_DIR}/github_repo_watcher.py >> {LOG_FILE} 2>&1") | crontab -
   ```

   **Windows (Task Scheduler):**
   ```batch
   schtasks /query /tn "GitHubRepoWatcher" >/dev/null 2>&1
   if errorlevel 1 (
     schtasks /create /tn "GitHubRepoWatcher" ^
       /tr "{PYTHON} {SCRIPT_DIR}\github_repo_watcher.py >> {LOG_FILE} 2>&1" ^
       /sc daily /st 09:00 /f
   )
   ```
   - `/sc daily /st 09:00` = 매일 오전 9시 (로컬 시간 기준)
   - 다른 주기: `/sc weekly /d MON` (매주 월요일)
   - 등록 확인: `schtasks /query /tn "GitHubRepoWatcher"`

8. 결과 보고: "✅ `{owner/repo}` 워치 등록 완료. SHA: `{sha[:7]}`"

**주의**: 첫 등록은 알림을 보내지 않는다. 현재 SHA를 베이스라인으로 기록만 함.

### list

1. `STATE_FILE` 읽기
2. 등록된 리포 없으면: "워치 중인 리포 없음. `/github-watch add owner/repo`로 추가하세요."
3. 있으면 테이블로 출력:
   ```
   | 리포 | 마지막 확인 | 현재 SHA | 상태 |
   |------|------------|---------|------|
   | owner/repo | 2026-04-12 | 4693c8f | ✅ 활성 |
   ```
4. 스케줄 상태 표시:
   - Linux/Mac: `crontab -l 2>/dev/null | grep github_repo_watcher`
   - Windows: `schtasks /query /tn "GitHubRepoWatcher" /fo LIST 2>nul`

### remove `<owner/repo>`

1. `STATE_FILE`에서 해당 리포 키 삭제
2. 저장
3. 남은 리포가 0개면 스케줄도 삭제:
   - Linux/Mac: `crontab -l | grep -v github_repo_watcher | crontab -`
   - Windows: `schtasks /delete /tn "GitHubRepoWatcher" /f`
4. "✅ `{owner/repo}` 워치 해제"

### test

1. `STATE_FILE`에서 첫 번째 리포 확인 (없으면 오류)
2. Telegram으로 테스트 메시지 전송:
   ```
   🔔 [테스트] GitHub Watcher 정상 작동 중
   📦 워치 중: {n}개 리포
   ⏰ 스케줄: {SCHEDULE}
   🖥️ OS: {OS}
   ```
3. 전송 성공 여부 확인 후 보고

---

## 스크립트 템플릿 (Events API + ETag 모드)

`setup` 실행 시 생성하는 `github_repo_watcher.py`.
**GitHub Events API + ETag 캐싱** 방식 — 변경 없으면 304 응답으로 rate limit 소모 없음.
5분 폴링 + 인증 토큰 = 사실상 무제한 실행 가능.

```python
#!/usr/bin/env python3
"""
GitHub Repo Watcher — Events API + ETag mode
새 PushEvent를 감지하면 Telegram 알림 전송.
ETag 캐싱으로 rate limit 소비 최소화 (304 응답 = 0 rate limit 소모).

State: ~/scripts/github_watch_state.json
Cron:  */5 * * * *  (5분마다 실행)
"""

import json, os, subprocess, sys, urllib.request, urllib.error
from datetime import datetime, timezone

SCRIPT_DIR   = os.path.dirname(os.path.abspath(__file__))
STATE_FILE   = os.path.join(SCRIPT_DIR, "github_watch_state.json")
TELEGRAM_ENV = os.path.expanduser("~/.claude/channels/telegram/.env")
TELEGRAM_CHAT_ID = ""  # skill add 시 채워줌

WATCHED_REPOS = []  # skill add로 추가됨


def get_github_token():
    """gh CLI에서 토큰 추출 (없으면 환경변수 폴백)."""
    try:
        result = subprocess.run(["gh", "auth", "token"],
                                capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            t = result.stdout.strip()
            if t:
                return t
    except Exception:
        pass
    return os.environ.get("GITHUB_TOKEN")


def load_telegram_token():
    try:
        with open(TELEGRAM_ENV) as f:
            for line in f:
                if line.startswith("TELEGRAM_BOT_TOKEN="):
                    return line.split("=", 1)[1].strip()
    except Exception:
        pass
    return None


def fetch_events(repo, etag, github_token):
    """Returns (events_or_None, new_etag, status_code)."""
    url = f"https://api.github.com/repos/{repo}/events?per_page=30"
    headers = {
        "User-Agent": "github-watcher/2.0",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
    }
    if github_token:
        headers["Authorization"] = f"Bearer {github_token}"
    if etag:
        headers["If-None-Match"] = etag

    req = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=15) as resp:
            return json.loads(resp.read()), resp.headers.get("ETag"), resp.status
    except urllib.error.HTTPError as e:
        if e.code == 304:
            return None, etag, 304
        print(f"[ERROR] GitHub API HTTP {e.code} for {repo}", file=sys.stderr)
        return [], etag, e.code
    except Exception as e:
        print(f"[ERROR] {repo}: {e}", file=sys.stderr)
        return [], etag, 0


def extract_push_commits(events, since_event_id, limit=5):
    new_commits, latest_event_id = [], since_event_id
    for event in events:
        if event.get("type") != "PushEvent":
            continue
        event_id = event.get("id")
        if since_event_id and event_id and event_id <= since_event_id:
            break
        if latest_event_id is None or (event_id and event_id > latest_event_id):
            latest_event_id = event_id
        payload = event.get("payload", {})
        branch = payload.get("ref", "").replace("refs/heads/", "")
        actor = event.get("actor", {}).get("login", "unknown")
        date = event.get("created_at", "")[:10]
        for c in payload.get("commits", []):
            new_commits.append({
                "sha": c.get("sha", "")[:7],
                "message": c.get("message", "").split("\n")[0][:80],
                "author": c.get("author", {}).get("name", actor),
                "branch": branch, "date": date,
            })
            if len(new_commits) >= limit:
                return new_commits, latest_event_id
    return new_commits, latest_event_id


def send_telegram(token, chat_id, text):
    url = f"https://api.telegram.org/bot{token}/sendMessage"
    payload = json.dumps({"chat_id": chat_id, "text": text,
                           "parse_mode": "HTML", "disable_web_page_preview": True}).encode()
    req = urllib.request.Request(url, data=payload,
                                  headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=15) as r:
            return json.loads(r.read()).get("ok", False)
    except Exception as e:
        print(f"[ERROR] Telegram: {e}", file=sys.stderr)
        return False


def main():
    tg_token = load_telegram_token()
    if not tg_token:
        sys.exit("[ERROR] TELEGRAM_BOT_TOKEN not found")
    tg_chat_id = TELEGRAM_CHAT_ID
    if not tg_chat_id:
        sys.exit("[ERROR] TELEGRAM_CHAT_ID not set")

    github_token = get_github_token()
    state = json.loads(open(STATE_FILE).read()) if os.path.exists(STATE_FILE) else {}
    now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
    updated = False

    for watched in WATCHED_REPOS:
        repo, name = watched["repo"], watched.get("name", watched["repo"].split("/")[1])
        print(f"[INFO] Checking {repo}...")
        repo_state = state.get(repo, {})
        etag, last_event_id = repo_state.get("etag"), repo_state.get("last_event_id")

        events, new_etag, status = fetch_events(repo, etag, github_token)
        if status == 304:
            print(f"[INFO] 304 Not Modified")
            state.setdefault(repo, {})["last_checked"] = now
            updated = True
            continue
        if not events or status >= 400:
            continue

        if last_event_id is None:
            latest_id = events[0]["id"] if events else None
            print(f"[INFO] First run. Recording event_id={latest_id}")
            state[repo] = {"last_event_id": latest_id, "etag": new_etag, "last_checked": now}
            updated = True
            continue

        new_commits, latest_event_id = extract_push_commits(events, last_event_id)
        state[repo] = {"last_event_id": latest_event_id or last_event_id,
                        "etag": new_etag or etag, "last_checked": now}
        updated = True

        if not new_commits:
            print(f"[INFO] No new push commits")
            continue

        n = len(new_commits)
        lines = "\n".join(
            f"  • [{c['sha']}] <code>{c['branch']}</code> {c['message']} ({c['author']}, {c['date']})"
            for c in new_commits
        )
        msg = (f"🔔 <b>{name}</b> 업데이트\n📦 <code>{repo}</code>\n\n"
               f"새 커밋 {n}개:\n{lines}\n\n"
               f"🔗 <a href=\"https://github.com/{repo}/commits\">커밋 히스토리</a>")
        state[repo]["last_notified_at"] = now
        if send_telegram(tg_token, tg_chat_id, msg):
            print(f"[INFO] Notified: {n} new commit(s)")

    if updated:
        with open(STATE_FILE, "w") as f:
            json.dump(state, f, indent=2, ensure_ascii=False)
    print(f"[INFO] Done.")


if __name__ == "__main__":
    main()
```
---

## 성공 기준

- [ ] `STATE_FILE`에 watched repo 엔트리 존재 (SHA 기록됨)
- [ ] 스케줄 등록 확인 (crontab 또는 Task Scheduler)
- [ ] `test` 서브커맨드로 Telegram 알림 수신 확인

---

## OS별 주의사항

### Linux/Mac 공통
- `chmod +x` 는 Linux/Mac에서만 실행 (Windows는 불필요)
- `~/logs/` 없으면 cron이 조용히 실패 → setup 시 반드시 생성
- cron 중복 방지: `grep github_repo_watcher` 먼저 확인

### Windows 전용
- Python 명령어: `python` 또는 `py` (시스템에 따라 다름, `python3`는 없을 수 있음)
- Task Scheduler는 로컬 시간 기준 (`/st 09:00` = 시스템 시간대 오전 9시)
- 로그 리다이렉션(`>> log 2>&1`)이 Task Scheduler에서 작동 안 할 수 있음 → 스크립트 내에서 직접 파일에 쓰는 방식으로 변경 가능
- `TELEGRAM_ENV` 경로는 `~/.claude/` 가 Windows에서 `C:\Users\{name}\.claude\`로 매핑됨 (`os.path.expanduser` 처리)
- Task Scheduler 작업 이름에 공백 불가 → `"GitHubRepoWatcher"` 형태 유지

### macOS 전용 (참고)
- macOS는 crontab 외 `launchd` (plist 방식) 사용 가능하지만 복잡함 → crontab으로 충분
- macOS Ventura+ 에서 crontab이 Full Disk Access 필요할 수 있음 → 시스템 환경설정에서 터미널 허용

---

## 공통 주의사항

- **첫 등록은 알림 없음**: SHA 베이스라인 기록만. 이후 변경부터 알림.
- **TELEGRAM_CHAT_ID**: `.env`에 없을 수 있음 → 기존 스크립트에서 하드코딩 값 확인
- **GitHub API 제한**: 인증 없이 시간당 60회. 일일 체크면 문제없음. 다수 리포 + 고빈도라면 `GITHUB_TOKEN` 헤더 추가:
  ```python
  headers={"User-Agent": "github-watcher/1.0", "Authorization": f"token {os.environ.get('GITHUB_TOKEN','')}"}
  ```
- **Private repo**: GitHub token 필수. public repo는 token 없이 작동.

---

## 확장 포인트

- `NOTIFIER=slack` → `SLACK_WEBHOOK_URL` 사용
- `NOTIFIER=discord` → `DISCORD_WEBHOOK_URL` 사용
- `NOTIFIER=stdout` → 크론 없이 로컬 테스트용
- 다수 리포 감시: `STATE_FILE`에 키만 추가하면 동일 스크립트/스케줄로 처리됨
