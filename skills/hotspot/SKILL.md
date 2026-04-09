---
name: hotspot
description: 코드 한 줄 읽기 전에 git 히스토리 분석으로 위험 파일·버스 팩터·팀 모멘텀을 진단한다. "위험한 파일 찾아줘", "hotspot 분석", "코드 읽기 전 분석" 등의 요청에 사용.
---

# Hotspot — Git 기반 코드베이스 위험도 진단

## 개요

코드를 열기 전에 **git 히스토리만으로** 프로젝트의 위험 지점을 파악한다.
5개 명령어를 실행하고 결과를 교차 분석해 위험도 리포트를 출력한다.

출처: [Git Commands Before Reading Code](https://piechowski.io/post/git-commands-before-reading-code/) — Ally Piechowski (2026-04-08)

---

## 실행 순서

### 0단계: git 레포 확인

현재 디렉토리가 git 레포인지 확인한다.

```bash
git rev-parse --show-toplevel 2>/dev/null || echo "NOT_A_GIT_REPO"
```

NOT_A_GIT_REPO가 나오면 사용자에게 git 레포 안에서 실행해달라고 안내하고 중단한다.

---

### 1단계: High-Churn 파일 (자주 바뀌는 파일)

```bash
git log --format=format: --name-only --since="1 year ago" \
  | sort | uniq -c | sort -nr | head -20
```

결과를 `CHURN_LIST`로 저장 (파일명 + 변경 횟수).

---

### 2단계: Bug 집중 파일

```bash
git log -i -E --grep="fix|bug|broken|hotfix|patch" \
  --name-only --format='' \
  | sort | uniq -c | sort -nr | head -20
```

결과를 `BUG_LIST`로 저장 (파일명 + bug 커밋 등장 횟수).

---

### 3단계: 버스 팩터

```bash
# 전체 히스토리
git shortlog -sn --no-merges | head -10

# 최근 6개월
git shortlog -sn --no-merges --since="6 months ago" | head -10
```

---

### 4단계: 프로젝트 모멘텀

```bash
git log --format='%ad' --date=format:'%Y-%m' | sort | uniq -c
```

---

### 5단계: 핫픽스·롤백 빈도

```bash
git log --oneline --since="1 year ago" \
  | grep -iE 'revert|hotfix|emergency|rollback' | wc -l

# 실제 커밋 목록도 확인
git log --oneline --since="1 year ago" \
  | grep -iE 'revert|hotfix|emergency|rollback' | head -10
```

---

## 분석 및 리포트 생성

### 교차 분석: Hotspot 파일 도출

`CHURN_LIST`와 `BUG_LIST`를 교차해 **두 리스트에 모두 등장하는 파일**을 찾는다.
→ 이 파일들이 진짜 위험 코드: 계속 바뀌고, 계속 버그가 나는 곳.

### 위험도 점수 계산 (각 파일)

```
위험도 = churn_rank_score + bug_rank_score
- churn_rank_score: 1위=20점, 2위=19점, ... 20위=1점
- bug_rank_score: 동일 방식
- 두 리스트 중 한 곳에만 있으면 해당 점수만 사용
```

### 버스 팩터 판정

- 1명이 전체 커밋의 60% 이상 → 🔴 CRITICAL
- 1명이 40~60% → 🟡 WARNING
- 분산 → 🟢 OK
- 전체 vs 최근 6개월 top contributor가 다르면 → 팀 이탈 가능성 언급

### 모멘텀 판정

최근 3개월 평균 vs 이전 3개월 평균 비교:
- 50% 이상 감소 → 🔴 모멘텀 급락
- 20~50% 감소 → 🟡 주의
- 유지/증가 → 🟢 OK

### 핫픽스 빈도 판정

- 연간 0~5건 → 🟢 정상
- 6~20건 → 🟡 주의 (배포 프로세스 점검 필요)
- 20건 이상 → 🔴 위험 (테스트/배포 파이프라인 문제)

---

## 출력 형식

다음 형식으로 텍스트 리포트를 출력한다:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 HOTSPOT REPORT — {repo_name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔴 위험 파일 Top N (churn + bug 교차)
┌─────────────────────────────────┬───────┬──────┬──────┐
│ 파일                             │ 위험도 │ churn│ bugs │
├─────────────────────────────────┼───────┼──────┼──────┤
│ src/components/SomeFile.tsx     │  35   │  18  │  17  │
└─────────────────────────────────┴───────┴──────┴──────┘

👤 버스 팩터
  전체: Alice (52%), Bob (23%), Carol (15%)
  최근 6개월: Bob (61%), Carol (28%)  ← Alice 이탈 신호

📈 모멘텀 (최근 6개월)
  Jan 42 ██████████
  Feb 38 █████████
  Mar 51 ████████████
  Apr 12 ███  ← 이달 (진행중)
  → 전 분기 대비 +12% 🟢

🚨 핫픽스/롤백 (최근 1년): 8건 🟡
  revert: fix broken auth (2026-03-12)
  hotfix: payment timeout (2026-02-28)
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 추천 액션
  1. src/components/SomeFile.tsx — 리뷰 전 담당자 확인 필수
  2. Bob이 최근 핵심 기여자 — 지식 공유 상태 확인
  3. 핫픽스 빈도 정상 범위, 배포 프로세스 무난함
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 주의사항

- **squash merge** 워크플로우에선 버스 팩터가 왜곡될 수 있음 (merge 커밋 기준이라 실제 저자 불명)
- **커밋 메시지 규율이 낮은 팀**에선 bug 집중 파일이 과소 탐지됨
- 결과가 비어있거나 이상하면 그 이유를 사용자에게 설명한다
- 리포트는 텍스트로 출력. 텔레그램 연동 환경이면 reply 툴로 전송 가능.
