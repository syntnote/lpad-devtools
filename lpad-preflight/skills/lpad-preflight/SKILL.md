---
name: lpad-preflight
description: 현재 디렉토리의 lpad 프로젝트가 배포 준비가 되었는지 사전 점검. 트랙(Amplify/ECS) 자동 감지, 파일 검증, 실제 빌드/린트 실행, 구체적 수정 가이드 제공. "lpad preflight", "배포 전 점검", "lpad-preflight" 키워드로 트리거.
disable-model-invocation: true
---

# lpad-preflight

<!-- SKILL_VERSION: 0.2.0 — 리포트 헤더 생성 시 이 값을 사용할 것.
     버전 변경 시 이 줄의 값과 ../../.claude-plugin/plugin.json의 version을 함께 bump. -->

lpad 인프라에 배포할 프로젝트를 사전 점검한다. lpad = launchpad(발사대)에서 착안한 "이륙 전 점검(preflight check)".

**목표:** 개발자가 `/lpad-preflight` 한 번 실행하면 자기 프로젝트의 배포 준비 상태를 종합 진단받는다.

## 판정 원칙 (엄격 모드)

- 검사 결과는 **`✅ PASS` 또는 `❌ FAIL` 둘 중 하나만** 존재한다. "경고", "보류", "주의", "스킵" 같은 중간 상태 없음.
- **모든 검사가 PASS여야 배포 가능.** 단 하나라도 FAIL이면 `❌ 실패`로 종료하고 배포 금지.
- 어떤 검사가 해당 프로젝트에 **적용 불가능**하면(예: Python 프로젝트에서 `tsc` 체크) 리포트에 넣지 않는다. 리포트에 나온 항목은 전부 실행됐고, 전부 PASS여야 한다.

## 실행 절차

반드시 이 순서대로 진행한다. 각 단계 결과를 누적해서 마지막에 종합 리포트를 만든다.

### 0단계: 실행 정보 캡처

리포트 헤더에 쓸 정보를 먼저 수집한다.

```bash
date '+%Y-%m-%d %H:%M:%S %Z'    # 실행 시각
pwd                              # 작업 디렉토리
```

버전은 이 SKILL.md 상단의 `SKILL_VERSION` 주석에서 읽어서 리포트에 표시한다 (현재: **0.2.0**).

### 1단계: 트랙 감지

현재 작업 디렉토리를 기준으로 프로젝트 구조를 파악한다.

```bash
ls -la
```

탐색 규칙:
- 루트에 `Dockerfile` 있으면 → **ECS 트랙**
- 루트에 `package.json`과 (`vite.config.*` 또는 `next.config.*` 또는 `webpack.config.*`)이 있으면 → **Amplify 트랙**
- 서브디렉토리(`api/`, `frontend/`, `landing/` 등)에 각각 위 조건 매치되면 → **모노레포** (각 서브디렉토리를 독립 프로젝트로 취급)
- 어느 쪽도 아니면 → "lpad 트랙 구조가 아님"으로 즉시 FAIL 종료

모노레포인 경우 각 서브 프로젝트마다 2~4단계를 반복 실행.

### 2단계: 정적 검사

파일을 Read/Grep으로 읽어 확인. 각 항목을 하나씩 명시적으로 검증한다. **적용 가능한 항목은 전부 PASS해야 한다.**

#### 공통 항목

**C1. `.gitignore` 민감 파일 제외** (적용: 항상)
- `.gitignore` Read
- `.env`, `.env.*`, `*.pem`, `*.key` 모두 포함되어야 함 → 하나라도 없으면 FAIL

**C2. 민감 파일 커밋 이력 없음** (적용: 항상)
- `git ls-files | grep -E '\.env$|\.env\.|firestore-key\.json|\.pem$|\.key$'`
- 결과 있으면 FAIL (파일명 제시). 파일 **내용**은 출력 금지.

**C3. 배포 워크플로우 존재** (적용: 항상)
- `.github/workflows/deploy-*.yml` 1개 이상 존재해야 함
- 없으면 FAIL ("인프라팀에 워크플로우 요청 필요")

**C4. main 직접 commit 없음** (적용: git 히스토리에 main 브랜치 있을 때)
- `git log main --oneline -20`에서 비-merge 직접 commit 확인
- merge commit이 아닌 직접 commit 발견 시 FAIL

#### ECS 트랙 항목

**E1. Dockerfile 존재**
- 없으면 FAIL

**E2. EXPOSE 포트와 앱 실행 포트 일치**
- Dockerfile `EXPOSE <port>` 값과 CMD/ENTRYPOINT의 port, 앱 코드(main.py, index.js 등)의 listen port가 **전부 동일**해야 함
- 불일치 시 FAIL — 어디가 어떤 값인지 구체적으로 제시

**E3. 헬스체크 엔드포인트 구현**
- lpad 기본 경로 `/api/health`
- 앱 코드에서 grep: `"/api/health"`, `'/api/health'`, `@app.get.*health`, `app.get.*health`
- 미발견 시 FAIL

**E4. `.dockerignore` 존재 + `.env` 제외**
- 파일 없거나 `.env`, `.env.*` 누락 시 FAIL (시크릿 이미지 포함 위험)
- `node_modules`, `__pycache__`, `.git`도 포함되어야 함. 빠지면 FAIL.

**E5. 비-root 사용자 설정**
- Dockerfile에 `USER <name>` 지시문 필수
- 없으면 FAIL

#### Amplify 트랙 항목

**A1. `package.json`의 `build` 스크립트**
- `scripts.build` 없으면 FAIL

**A2. 빌드 설정 파일 존재**
- `vite.config.*`, `next.config.*`, `webpack.config.*` 중 하나 존재
- 없으면 FAIL

**A3. lockfile 존재**
- `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` 중 하나 필수
- 없으면 FAIL (CI에서 install 실패)

### 3단계: 실행 검사

**실행 전 안내:** "빌드와 테스트를 실제로 실행합니다 (2~5분 소요)" 사용자에게 알리고 진행.

각 명령은 Bash로 직접 실행. timeout 설정 필수. **사전 조건 위반은 FAIL** (예: Docker 필요한 프로젝트인데 Docker 미설치 → "Docker가 필요합니다" FAIL).

#### ECS 트랙 실행

**R-E1. Docker 빌드**
```bash
docker build -t lpad-preflight-test:latest . 2>&1 | tail -50
```
- timeout 600000
- 실패 또는 `docker` 명령 없으면 FAIL

**R-E2. 컨테이너 헬스체크**
```bash
CID=$(docker run -d -p 0:<container_port> lpad-preflight-test:latest)
HOST_PORT=$(docker port $CID <container_port> | head -1 | sed 's/.*://')

HTTP_CODE=000
for i in 1 2 3 4 5; do
  sleep 2
  HTTP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" "http://localhost:${HOST_PORT}/api/health" 2>/dev/null || echo 000)
  [[ "$HTTP_CODE" == "200" ]] && break
done

if [[ "$HTTP_CODE" != "200" ]]; then
  docker logs $CID 2>&1 | tail -30
fi

docker rm -f $CID >/dev/null 2>&1 || true
docker rmi lpad-preflight-test:latest >/dev/null 2>&1 || true
```
- 200 아니면 FAIL (로그 제시)
- 정리(`rm`, `rmi`)는 FAIL 상황에서도 반드시 실행

#### Amplify 트랙 실행

**R-A1 + R-A2. 의존성 설치 + 빌드**

```bash
if [[ -f pnpm-lock.yaml ]]; then
  PM="pnpm"; INSTALL="pnpm install --frozen-lockfile"; BUILD="pnpm run build"
elif [[ -f yarn.lock ]]; then
  PM="yarn"; INSTALL="yarn install --frozen-lockfile"; BUILD="yarn run build"
elif [[ -f package-lock.json ]]; then
  PM="npm"; INSTALL="npm ci"; BUILD="npm run build"
fi

$INSTALL                              # timeout 300000
$BUILD 2>&1 | tail -40                # timeout 600000
ls dist/ 2>/dev/null || ls build/ 2>/dev/null || ls .next/ 2>/dev/null | head
```

- install 실패, 빌드 실패, output 디렉토리가 비어있으면 전부 FAIL

#### 공통 실행 (해당 도구/설정이 있는 경우에만 실행)

Node 명령은 앞서 감지한 `$PM`을 재사용.

**R-C1. 린트** (적용: 설정 있을 때)
- `package.json`의 `scripts.lint` 있으면 `$PM run lint` → 실패 시 FAIL
- Python `ruff.toml`, `.ruff.toml`, `pyproject.toml`에 ruff 설정 있으면 `ruff check .` → 실패 시 FAIL
- `.flake8`, `setup.cfg`에 flake8 설정 있으면 `flake8 .` → 실패 시 FAIL

**R-C2. 타입 체크** (적용: 설정 있을 때)
- `package.json`의 `scripts.typecheck` → `$PM run typecheck` → 실패 시 FAIL
- `tsconfig.json` 있으면 `npx tsc --noEmit` → 실패 시 FAIL
- `mypy.ini` 또는 `pyproject.toml`에 mypy 설정 있으면 `mypy .` → 실패 시 FAIL
- `pyrightconfig.json` 있으면 `pyright` → 실패 시 FAIL

**R-C3. 테스트** (적용: 테스트 코드 있을 때)
- `package.json`의 `scripts.test`가 placeholder(`"echo ... exit 1"`) 아닌 경우 → `$PM test` → 실패 시 FAIL
- `tests/` 디렉토리 + `pytest` 설정이 있으면 `pytest` → 실패 시 FAIL
- timeout 300000

**적용 가능 판단 원칙:**
- "설정 파일이 있다" = 개발자가 이 검사를 기대한다 = 반드시 실행 + PASS 필수
- 설정 파일이 아예 없으면 해당 검사는 리포트에 포함하지 않는다 (적용 불가)

### 4단계: 리포트 생성

**리포트 최상단에 반드시** 버전과 실행 시각을 포함한다.

```
🚀 lpad-preflight v<SKILL_VERSION>
실행 시각: <YYYY-MM-DD HH:MM:SS TZ>
프로젝트: <프로젝트명 또는 디렉토리명>
트랙: <ECS | Amplify | 모노레포>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[정적 검사]
  ✅ <항목명>
  ❌ <항목명>
     💡 <구체적 수정 방법>
  ...

[빌드 검사]
  ✅ <항목명> (<소요시간>)
  ❌ <항목명> (<소요시간>)
     💡 <로그 요약 or 수정 방법>
  ...

[코드 품질]
  ✅ <도구명>: <결과>
  ❌ <도구명>: <결과 요약>
     💡 <수정 방법>
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
결과: <✅ 통과 | ❌ 실패> (FAIL N건 / 전체 M건)

<다음 단계 안내>
```

**상태 판정 (엄격 이진):**
- 모든 검사 PASS → `✅ 통과` — "배포 가능"
- 하나라도 FAIL → `❌ 실패` — "FAIL 항목을 모두 해결한 뒤 재실행하세요. 현 상태로 배포 금지."

**아이콘 규칙 (단 두 개):**
- ✅ PASS
- ❌ FAIL

⚠️, ⏭️, 기타 상태 아이콘은 **절대 사용 금지**.

## 출력 원칙

1. **거짓말 금지** — 실행 안 한 검사는 리포트에 넣지 않는다. 리포트에 나온 항목은 전부 실제로 실행된 것이다.
2. **엄격 이진** — PASS/FAIL 외 중간 상태는 존재하지 않는다. "경고", "주의", "나중에 봐도 됨" 같은 표현 금지.
3. **구체적 수정 가이드** — "경로 틀림" ❌ / "Dockerfile:12의 EXPOSE 3000을 8080으로 수정" ✅
4. **파일:라인 인용** — 코드 참조 시 `path:line` 형식
5. **긴 에러는 요약** — 빌드 실패 시 마지막 30~50줄만
6. **정리 책임** — `docker run`한 컨테이너는 반드시 `docker rm -f`, `docker build`로 만든 `lpad-preflight-test:latest` 이미지도 `docker rmi`로 정리
7. **버전/시각 헤더 필수** — 리포트 최상단에 `lpad-preflight v<SKILL_VERSION>`과 실행 시각을 반드시 기록

## 주의사항

- 민감 파일 검사 시 **파일 내용을 출력하지 말 것**. 파일명과 존재 여부만.
- Docker가 필요한 ECS 프로젝트인데 `docker` 명령이 없거나 데몬이 안 떠있으면 R-E1/R-E2는 FAIL로 처리 ("Docker Desktop 실행 필요" 안내 포함).
- 모노레포에서 하위 프로젝트 검사 시 Bash `cd`로 디렉토리 이동.
- 검사 도중 오류가 나도 가능한 한 나머지 검사 진행 (치명적 오류만 조기 종료). 진행된 검사들 전부 PASS여야 전체 PASS.

## 흔한 lpad 오류 패턴 (진단 시 참고)

| 증상 | 실제 원인 |
|------|----------|
| Amplify 빌드 성공했는데 API 호출 실패 | `VITE_API_URL` 미설정 → `/api`로 상대경로 fallback |
| ECS 배포는 성공했는데 서비스 안 됨 | 헬스체크 경로 불일치 또는 200 미반환 |
| CORS 에러 | 백엔드 `ALLOWED_ORIGINS`에 프론트 도메인 누락 |
| 빌드는 되는데 런타임 오류 | `.env.local` 등 로컬 전용 파일에 의존 |
