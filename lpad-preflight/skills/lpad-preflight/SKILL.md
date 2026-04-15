---
name: lpad-preflight
description: 현재 디렉토리의 lpad 프로젝트가 배포 준비가 되었는지 사전 점검. 트랙(Amplify/ECS) 자동 감지, 파일 검증, 실제 빌드/린트 실행, 구체적 수정 가이드 제공. "lpad preflight", "배포 전 점검", "lpad-preflight" 키워드로 트리거.
version: 0.1.0
---

# lpad-preflight

lpad 인프라에 배포할 프로젝트를 사전 점검한다. lpad = launchpad(발사대)에서 착안한 "이륙 전 점검(preflight check)".

**목표:** 개발자가 `/lpad-preflight` 한 번 실행하면 자기 프로젝트의 배포 준비 상태를 종합 진단받는다.

## 실행 절차

반드시 이 순서대로 진행한다. 각 단계 결과를 누적해서 마지막에 종합 리포트를 만든다.

### 1단계: 트랙 감지

현재 작업 디렉토리(`pwd`)를 기준으로 프로젝트 구조를 파악한다.

```bash
pwd
ls -la
```

탐색 규칙:
- 루트에 `Dockerfile` 있으면 → **ECS 트랙**
- 루트에 `package.json`과 (`vite.config.*` 또는 `next.config.*` 또는 `webpack.config.*`)이 있으면 → **Amplify 트랙**
- 서브디렉토리(`api/`, `frontend/`, `landing/` 등)에 각각 위 조건 매치되면 → **모노레포** (각 서브디렉토리를 독립 프로젝트로 취급)
- 어느 쪽도 아니면 → "lpad 트랙 구조가 아님" 경고 후 종료

모노레포인 경우 각 서브 프로젝트마다 2~4단계를 반복 실행.

### 2단계: 정적 검사

파일을 Read/Grep으로 읽어 확인. 절대 건너뛰지 말 것 — 각 항목을 하나씩 명시적으로 검증한다.

#### 공통 항목

**C1. `.gitignore` 민감 파일 제외**
- `.gitignore` 파일 Read
- 다음 패턴이 모두 포함되어야 함: `.env`, `.env.*`, `*.pem`, `*.key`
- 하나라도 없으면 **필수 실패**

**C2. 민감 파일 커밋 이력 확인**
- `git ls-files | grep -E '\.env$|\.env\.|firestore-key\.json|\.pem$|\.key$'` 실행
- 결과가 있으면 **필수 실패** (해당 파일명 제시)
- 커밋되지 않은 경우도 확인: `git log --all --full-history --source -- .env .env.* 2>/dev/null | head -5`

**C3. 배포 워크플로우 존재**
- `.github/workflows/` 디렉토리 확인
- `deploy-*.yml` 파일 1개 이상 존재해야 함
- 없으면 **경고** ("인프라팀에 워크플로우 요청 필요")

**C4. main 브랜치 직접 push 이력**
- `git log main --oneline -20` 실행
- PR merge 커밋이 아닌 직접 커밋이 최근 있으면 **경고**
- (Merge commit은 `^Merge pull request` 또는 `squash merge`의 경우 이 검사 생략 가능)

#### ECS 트랙 항목

**E1. Dockerfile 존재**
- Read `Dockerfile` — 없으면 **필수 실패**

**E2. EXPOSE 포트와 앱 실행 포트 일치**
- Dockerfile에서 `EXPOSE <port>` 추출
- Dockerfile의 `CMD`/`ENTRYPOINT`에서 포트 추출 (예: `--port 8080`, `uvicorn ... --port 8080`)
- 앱 코드(main.py, index.js 등)에서도 포트 사용 패턴 grep (`PORT`, `listen(`, `--port`)
- 불일치하면 **필수 실패** — 구체적으로 어디가 몇 번인지 제시

**E3. 헬스체크 엔드포인트 구현 확인**
- lpad 등록 시 기본 경로 `/api/health` (main.tf에서 확인 가능)
- 앱 코드에서 해당 경로 grep: `"/api/health"`, `'/api/health'`, `@app.get.*health`, `app.get.*health`
- 미발견 시 **필수 실패**
- 발견되면 해당 파일과 라인 제시

**E4. `.dockerignore` 존재 + 민감 파일 제외**
- `.dockerignore` Read — 없으면 **필수 실패**
- `.env`, `.env.*` 포함 여부 — 빠지면 **필수 실패** (시크릿이 이미지에 포함되는 보안 사고 위험)
- `node_modules`, `__pycache__`, `.git` 포함 — 빠지면 **경고** (빌드 효율 저하)

**E5. 비-root 사용자 설정**
- Dockerfile에 `USER <name>` 지시문 존재 여부
- 없으면 **경고** (필수 아니지만 보안 권장)

#### Amplify 트랙 항목

**A1. `package.json`의 `build` 스크립트**
- `package.json` Read → `scripts.build` 존재 여부
- 없으면 **필수 실패**

**A2. 빌드 설정 파일 존재**
- `vite.config.*`, `next.config.*`, `webpack.config.*` 중 하나 존재
- 없으면 **경고** (감지된 프레임워크로 동작할 수 있으나 설정 명시 권장)

**A3. lockfile 존재**
- `package-lock.json` 또는 `pnpm-lock.yaml` 또는 `yarn.lock` 존재
- 없으면 **필수 실패** (CI에서 `npm ci` 실패)

### 3단계: 실행 검사

**실행 전 안내:** "빌드와 린트를 실제로 실행합니다 (2~5분 소요)" 사용자에게 알리고 진행.

각 명령은 Bash로 직접 실행. timeout 설정 필수.

#### ECS 트랙 실행

**R-E1. Docker 빌드**
```bash
docker build -t lpad-preflight-test:latest . 2>&1 | tail -50
```
- timeout 600000 (10분)
- 실패 시 **필수 실패**. 마지막 에러 줄 제시
- 성공 시 다음으로

**R-E2. 컨테이너 실행 + 헬스체크**
- 등록 포트로 컨테이너 실행 (디폴트 3000, 보통 `Dockerfile`의 EXPOSE 값 사용)
```bash
# 랜덤 호스트 포트로 충돌 회피
CID=$(docker run -d -p 0:<container_port> lpad-preflight-test:latest)
sleep 3
HOST_PORT=$(docker port $CID <container_port> | awk -F: '{print $2}' | head -1)
# 헬스체크
curl -sf -o /dev/null -w "%{http_code}" "http://localhost:${HOST_PORT}/api/health"
# 정리
docker rm -f $CID
```
- 200이 아니면 **필수 실패**
- 실패 시 컨테이너 로그 제시: `docker logs $CID 2>&1 | tail -30`
- 정리(`docker rm -f`)는 실패해도 반드시 실행

#### Amplify 트랙 실행

**R-A1. 의존성 설치**
```bash
# lockfile에 따라 분기
[[ -f pnpm-lock.yaml ]] && pnpm install --frozen-lockfile
[[ -f yarn.lock ]] && yarn install --frozen-lockfile
[[ -f package-lock.json ]] && npm ci
```
- timeout 300000 (5분)
- 실패 시 **필수 실패**

**R-A2. 빌드**
```bash
npm run build 2>&1 | tail -40
```
- timeout 600000 (10분)
- 실패 시 **필수 실패**
- 성공 시 `ls dist/ 2>/dev/null | head` 또는 설정된 outDir 확인
- `dist/`가 비어있으면 **필수 실패**

#### 공통 실행 (있으면 실행, 없으면 스킵)

**R-C1. 린트**
- `package.json`의 `scripts.lint` 존재 시 `npm run lint`
- Python: `ruff check .` 또는 `flake8 .` (둘 중 설정 파일 있는 것)
- 실패 시 **경고** (배포 차단 아님)

**R-C2. 타입 체크**
- `package.json`의 `scripts.typecheck` 또는 `tsc --noEmit` (tsconfig.json 있을 때)
- `mypy` 또는 `pyright` (설정 파일 있을 때)
- 실패 시 **경고**

**R-C3. 테스트**
- `npm test` (scripts.test 있고 `"test": "echo ..."` 같은 플레이스홀더 아닐 때)
- `pytest` (tests/ 디렉토리 존재 시)
- timeout 300000 (5분)
- 실패 시 **경고** (배포는 가능하나 권장 안 함)

### 4단계: 리포트 생성

아래 형식으로 종합 리포트 출력. 아이콘, 구분선 포함. 섹션은 실제 실행한 것만 포함.

```
🚀 lpad-preflight 진단 결과
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

프로젝트: <프로젝트명>
트랙: <ECS | Amplify | 모노레포>
검사 시각: <timestamp>

[정적 검사]
  <아이콘> <항목명>
  <실패/경고 시 💡 구체적 수정 방법>
  ...

[빌드 검사]
  <아이콘> <항목명> (<소요시간>)
  ...

[코드 품질]
  <아이콘> <도구명>: <결과>
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
배포 준비 상태: <✅ 통과 | ⚠️ 주의 | ❌ 실패>
요약: <필수 실패 N건, 경고 M건>

<다음 단계 안내>
```

**상태 판정:**
- 필수 실패 0건, 경고 0건 → `✅ 통과` — "배포 가능합니다"
- 필수 실패 0건, 경고 >0 → `⚠️ 주의` — "배포 가능하지만 개선 권장"
- 필수 실패 >0 → `❌ 실패` — "필수 항목을 먼저 해결하세요"

**아이콘 규칙:**
- ✅ 통과
- ❌ 필수 실패
- ⚠️ 경고
- ⏭️ 스킵됨 (도구 미설치 등)

## 출력 원칙

1. **거짓말 금지** — 실행 안 한 검사는 리포트에 포함 금지. 스킵된 건 ⏭️로 표시.
2. **구체적 수정 가이드** — "경로 틀림" ❌ / "Dockerfile:12의 EXPOSE 3000을 8080으로 수정" ✅
3. **파일:라인 인용** — 코드 참조 시 `path:line` 형식
4. **긴 에러는 요약** — 빌드 실패 시 마지막 30~50줄만
5. **정리 책임** — `docker run`한 컨테이너는 반드시 `docker rm -f`로 정리

## 주의사항

- 민감 파일 검사 시 **파일 내용을 출력하지 말 것**. 파일명과 존재 여부만 리포트.
- `docker` 명령이 없는 환경(Docker 미설치)이면 R-E1/R-E2는 스킵하고 ⏭️로 표시. "Docker Desktop 실행 필요" 안내.
- 모노레포에서 하위 프로젝트 검사 시 디렉토리 전환은 Bash `cd` 사용(`working_directory` 파라미터 활용).
- 검사 중 오류가 나도 가능한 한 나머지 검사 진행. 치명적 오류(디렉토리 접근 불가 등)만 조기 종료.

## 흔한 lpad 오류 패턴 (참고)

리포트 작성 시 관련된 경우 이 패턴도 점검:

| 증상 | 실제 원인 |
|------|----------|
| Amplify 빌드 성공했는데 API 호출 실패 | `VITE_API_URL` 미설정 → `/api`로 상대경로 fallback |
| ECS 배포는 성공했는데 서비스 안 됨 | 헬스체크 경로 불일치 또는 200 미반환 |
| CORS 에러 | 백엔드 `ALLOWED_ORIGINS`에 프론트 도메인 누락 |
| 빌드는 되는데 런타임 오류 | `.env.local` 등 로컬 전용 파일에 의존 |
