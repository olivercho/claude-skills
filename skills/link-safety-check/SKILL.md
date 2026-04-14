---
name: link-safety-check
description: |
  외부 링크(GitHub 레포, npm/pip 패키지, 웹 URL)의 보안 위험을 정적 분석으로 체크.
  Use when: 외부 링크 설치/방문 전 안전 여부 확인. 트리거: "위험성 체크", "안전한가", "써도 돼?", URL/GitHub 링크 단독 전송.
user_invocable: true
allowed-tools:
  - Bash
  - Read
---

# link-safety-check — 외부 링크 보안 위험 분석

## 목적

GitHub 레포 / npm·pip 패키지 / 웹 URL을 코드 실행 없이 정적 분석하여
SAFE / CAUTION / RISKY 판정과 근거를 제공한다.

## 트리거

- "이거 위험성 체크해줘"
- "써도 돼?", "안전한가?", "설치해도 돼?"
- URL 또는 GitHub 링크를 단독으로 보낼 때
- `/link-safety-check https://...`

## 입력

- URL 1개 이상

## 단계

### Step 0: 링크 타입 감지

```
github.com/{owner}/{repo}       → GitHub 레포
npmjs.com/package/{name}        → npm 패키지
pypi.org/project/{name}         → pip 패키지
그 외                            → 웹 URL
```

### Step 1: GitHub 레포 체크

#### 1-A. 메타데이터 (GitHub API, 인증 불필요)

```bash
curl -s "https://api.github.com/repos/{owner}/{repo}" -o /tmp/repo_meta.json
python3 -c "
import json
d = json.load(open('/tmp/repo_meta.json'))
print('stars:', d.get('stargazers_count'))
print('forks:', d.get('forks_count'))
print('created:', d.get('created_at'))
print('updated:', d.get('updated_at'))
print('description:', d.get('description'))
print('language:', d.get('language'))
print('open_issues:', d.get('open_issues_count'))
"
```

위험 신호:
- 스타 수가 많은데 생성일이 최근 (< 30일) → 가짜 인기 가능성
- 업데이트가 3년 이상 없음 → 방치된 레포 (보안 패치 없음)

#### 1-B. 설치 스크립트 위험 패턴

주요 파일 다운로드 후 검사:
```bash
for FILE in README.md install.sh setup.py package.json requirements.txt Makefile; do
  curl -sL "https://raw.githubusercontent.com/{owner}/{repo}/HEAD/$FILE" -o "/tmp/check_$FILE" 2>/dev/null
done
```

grep 패턴 (각 파일에 적용):
```bash
# 원격 실행 패턴 (가장 위험)
grep -En "curl.*(sh|bash)|wget.*(sh|bash)|bash.*<\(curl|bash.*<\(wget|sh.*<\(curl" /tmp/check_* 2>/dev/null

# base64 디코딩 실행
grep -En "base64.*-d.*\|.*sh|echo.*base64.*exec" /tmp/check_* 2>/dev/null

# postinstall 자동 실행 (package.json)
python3 -c "
import json, sys
try:
    d = json.load(open('/tmp/check_package.json'))
    scripts = d.get('scripts', {})
    for k in ['postinstall', 'preinstall', 'install']:
        if k in scripts:
            print(f'⚠ {k}:', scripts[k])
except: pass
" 2>/dev/null

# 외부 IP/도메인 하드코딩
grep -Eon "([0-9]{1,3}\.){3}[0-9]{1,3}:[0-9]+" /tmp/check_* 2>/dev/null | head -5
```

#### 1-C. 최근 커밋 이력

```bash
curl -s "https://api.github.com/repos/{owner}/{repo}/commits?per_page=5" | \
  python3 -c "
import json, sys
commits = json.load(sys.stdin)
for c in commits:
    print(c['commit']['author']['date'][:10], c['commit']['message'][:80])
"
```

갑작스러운 대량 변경, 의미 없는 커밋 메시지는 공급망 공격 신호.

### Step 2: npm 패키지 체크

```bash
PKG="{package_name}"
# 패키지 메타데이터
curl -s "https://registry.npmjs.org/$PKG/latest" -o /tmp/npm_meta.json
python3 -c "
import json
d = json.load(open('/tmp/npm_meta.json'))
scripts = d.get('scripts', {})
print('version:', d.get('version'))
print('author:', d.get('author'))
print('install scripts:', {k:v for k,v in scripts.items() if 'install' in k})
"

# OSV.dev CVE 조회
curl -s -X POST "https://api.osv.dev/v1/query" \
  -H "Content-Type: application/json" \
  -d "{\"package\":{\"name\":\"$PKG\",\"ecosystem\":\"npm\"}}" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); vulns=d.get('vulns',[]); print(f'{len(vulns)} CVEs found'); [print(' -', v.get('id'), v.get('summary','')[:80]) for v in vulns[:5]]"
```

위험 신호:
- `postinstall` / `preinstall` 스크립트 존재
- 알려진 CVE 다수
- 패키지명이 유명 패키지와 1~2자 차이 (타이포스쿼팅)

### Step 3: pip 패키지 체크

```bash
PKG="{package_name}"
# PyPI 메타데이터
curl -s "https://pypi.org/pypi/$PKG/json" | python3 -c "
import json, sys
d = json.load(sys.stdin)['info']
print('version:', d.get('version'))
print('author:', d.get('author'))
print('home_page:', d.get('home_page'))
"

# OSV.dev CVE 조회
curl -s -X POST "https://api.osv.dev/v1/query" \
  -H "Content-Type: application/json" \
  -d "{\"package\":{\"name\":\"$PKG\",\"ecosystem\":\"PyPI\"}}" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); vulns=d.get('vulns',[]); print(f'{len(vulns)} CVEs found'); [print(' -', v.get('id'), v.get('summary','')[:80]) for v in vulns[:5]]"
```

### Step 4: 웹 URL 체크

```bash
URL="{url}"

# 리다이렉트 체인 확인
curl -s -I -L --max-redirs 10 -o /dev/null -w "Final URL: %{url_effective}\nHTTP code: %{http_code}\nRedirects: %{num_redirects}\nSSL verify: %{ssl_verify_result}\n" "$URL"

# 도메인 타이포스쿼팅 의심 패턴 (시각적으로 확인)
python3 -c "
from urllib.parse import urlparse
import re
u = urlparse('$URL')
domain = u.netloc
# 숫자로 알파벳 대체 (1→l, 0→o 등)
suspicious = bool(re.search(r'[0-9]', domain.split('.')[0]))
# 서브도메인이 3개 이상
deep = domain.count('.') >= 3
print('domain:', domain)
print('숫자 포함 (타이포스쿼팅 의심):', suspicious)
print('깊은 서브도메인 (3단계+):', deep)
print('HTTP (암호화 없음):', u.scheme == 'http')
"
```

### Step 5: 판정 및 리포트

수집한 증거를 종합해 아래 형식으로 출력:

```
## 링크 안전성 분석: {링크}

**판정: ✅ SAFE / ⚠️ CAUTION / 🔴 RISKY**

### 체크 결과
- [✅/⚠️/🔴] 항목: 설명
- [✅/⚠️/🔴] 항목: 설명
...

### 근거 요약
[2-3줄]

### 권장 행동
[설치/방문 전 추가 확인 사항 또는 "안전하게 사용 가능"]
```

## 판정 기준

| 판정 | 기준 |
|------|------|
| ✅ SAFE | 위험 신호 없음, 출처 신뢰 가능, CVE 없음 |
| ⚠️ CAUTION | 낮은 활동/스타 수, 주의 필요하나 명확한 악성 신호 없음 |
| 🔴 RISKY | curl\|bash 패턴, 알려진 CVE, 타이포스쿼팅 의심, 가짜 인기 |

## 성공 기준

- 링크 타입 자동 감지됨
- 타입에 맞는 체크 항목 전부 실행
- SAFE / CAUTION / RISKY 판정 + 근거 제시
- 코드 실행 없이 정적 분석만으로 완료

## 주의사항 (Pitfalls)

### ⚠️ GitHub API rate limit
인증 없이 시간당 60 요청. 대부분의 경우 충분하지만 연속 분석 시 초과 가능.
초과 시: `curl -s "https://api.github.com/rate_limit"` 로 현황 확인.

### ⚠️ Private 레포는 분석 불가
GitHub API가 404 반환. 로컬 클론 후 수동 분석 필요.

### ⚠️ 이 스킬은 정적 분석만 수행
- 동적 분석(실제 실행) 불가
- 100% 안전 보장 불가 — 판단 보조 도구
- SAFE 판정이 절대 안전을 의미하지 않음

### ⚠️ 단축 URL (bit.ly 등)은 Step 4 리다이렉트 체인으로 먼저 최종 URL 확인
최종 URL이 의심스러우면 해당 타입으로 재분석.
