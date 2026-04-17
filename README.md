# lpad-devtools

신트노트 **lpad** 플랫폼 개발자를 위한 통합 도구 모음. lpad 위에서 프로젝트를 만들고 배포하고 운영하는 사람들이 공통으로 쓰는 것들을 한 레포에서 관리합니다.

현재는 **Claude Code 플러그인**(`lpad`)이 주 구성이며, 앞으로 프로젝트 템플릿·공용 GitHub Actions·부트스트랩 스크립트 등으로 범위를 넓혀갑니다.

## 제공 항목

| 카테고리 | 이름 | 제공 커맨드 | 상태 |
|----------|------|-------------|------|
| Claude Code 플러그인 | [`lpad`](#claude-code-플러그인-lpad) | `/lpad-preflight` | v0.5.0 |
| 프로젝트 템플릿 | — | — | 예정 |
| 공용 GitHub Actions | — | — | 예정 |

---

## Claude Code 플러그인 `lpad`

### 설치

Claude Code를 연 상태에서 아래를 순서대로 입력하면 끝. 별도 인증·환경변수·PAT 전부 불필요 (public repo).

```
/plugin marketplace add syntnote/lpad-devtools
```

```
/plugin install lpad@syntnote --scope user
```

> **`--scope user` 필수.** `lpad`는 인프라팀이 정의한 배포 규정을 검증하는 도구이므로, 모든 프로젝트가 **동일한 최신 버전**을 공유해야 합니다. `local`/`project` scope로 설치하면 프로젝트마다 규정 기준이 갈라져 배포 파이프라인(항상 최신 기준)과 괴리가 생깁니다.

이후 Claude Code 시작 시마다 자동으로 최신 버전 업데이트. 앞으로 `/lpad-doctor`, `/lpad-new` 같은 새 커맨드가 추가되어도 **이 플러그인 하나**로 같이 깔립니다 — 따로 설치할 필요 없음.

### 기존 사용자 마이그레이션 (v0.4 → v0.5)

v0.5부터 **marketplace 이름**과 **플러그인 이름**이 바뀌었습니다 (`syntnote-lpad` → `syntnote`, `lpad-preflight` → `lpad`). 기존 설치를 정리하고 새 이름으로 재등록하세요:

```
/plugin uninstall lpad-preflight@syntnote-lpad
```

```
/plugin marketplace remove syntnote-lpad
```

```
/plugin marketplace add syntnote/lpad-devtools
```

```
/plugin install lpad@syntnote --scope user
```

**검증:** 임의 디렉토리에서 `/lpad-preflight` 실행 → 리포트 헤더의 버전이 위 표의 최신과 일치하면 OK.

### 잘못 설치했다면 (복구)

`local`/`project` scope로 박아뒀거나 여러 scope에 중복 등록된 경우 구버전이 고착돼 업데이트가 안 될 수 있습니다. 전부 uninstall 후 `--scope user`로 재설치하세요:

```
/plugin uninstall lpad@syntnote
```

```
/plugin install lpad@syntnote --scope user
```

uninstall은 scope별로 복수 실행될 수 있습니다. 전부 제거 후 재설치.

### 제공 커맨드

#### `/lpad-preflight`

자기 프로젝트 디렉토리에서 실행. lpad 배포 전 사전 점검.

- 트랙(Amplify/ECS) 자동 감지 (모노레포 지원)
- 기본 정적 검사: `Dockerfile`, `.gitignore`, `.dockerignore`, 헬스체크 엔드포인트, 포트 일치, 작업 트리 clean, origin 동기화
- **환경변수·URL·비밀 검증**: `.env.example`과 코드 참조 env 대조, 하드코딩 URL, Vite `process.env` 오용, 프론트 번들 비밀 노출, `.env.local` 의존
- **백엔드 안정성**: CORS 설정, 헬스체크의 외부 의존성 검출
- **빌드·테스트 인프라**: 테스트 스크립트/파일 존재, Dockerfile 멀티스테이지, CI 빌드·테스트 스텝 포함
- 실행 검사: `docker build` + 컨테이너 헬스체크, `npm ci`/`build`, 린트/타입/테스트 실제 실행
- 엄격 이진 판정(PASS/FAIL만) + 구체적 수정 가이드 + 파일:라인 인용
- 실행 시 자체 버전 체크 — 구버전 사용 시 업데이트 안내
