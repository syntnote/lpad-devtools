# lpad-devtools

신트노트 lpad 개발자용 도구·확장 모음. 현재는 Claude Code 플러그인 마켓플레이스로 운영되며, 향후 다른 개발자 도구(템플릿, 스크립트 등)도 이 레포에 함께 관리합니다.

## 설치 (개발자용)

Claude Code를 연 상태에서 아래를 순서대로 입력하면 끝. 별도 인증·환경변수·PAT 전부 불필요 (public repo).

```
/plugin marketplace add syntnote/lpad-devtools
```

```
/plugin install lpad-preflight@syntnote-lpad --scope user
```

> **`--scope user` 필수.** lpad 플러그인은 인프라팀이 정의한 배포 규정을 검증하는 도구이므로, 모든 프로젝트가 **동일한 최신 버전**을 공유해야 합니다. `local`/`project` scope로 설치하면 프로젝트마다 규정 기준이 갈라져 배포 파이프라인(항상 최신 기준)과 괴리가 생깁니다.

이후 Claude Code 시작 시마다 자동으로 최신 버전 업데이트.

### 잘못 설치했다면 (복구)

과거에 scope 명시 없이 설치했거나 `local`/`project` scope로 박아둔 경우, 구버전이 고착돼 업데이트가 안 될 수 있습니다. 다음 순서로 재설치:

```
/plugin uninstall lpad-preflight@syntnote-lpad
```

```
/plugin install lpad-preflight@syntnote-lpad --scope user
```

같은 플러그인이 여러 scope에 중복 등록된 경우 uninstall이 항목별로 반복될 수 있습니다. 모두 제거 후 재설치하세요. 검증: 임의 디렉토리에서 `/lpad-preflight` 실행 시 리포트 헤더의 버전이 아래 표의 최신과 일치해야 합니다.

## 플러그인 목록

| 이름 | 용도 | 버전 |
|------|------|------|
| `lpad-preflight` | lpad 배포 전 프로젝트 사전 점검 (엄격 이진 판정 + 버전 자체 체크) | 0.3.0 |

### lpad-preflight

자기 프로젝트 디렉토리에서:

```
/lpad-preflight
```

- 트랙(Amplify/ECS) 자동 감지
- Dockerfile, `.gitignore`, 헬스체크 등 정적 검사
- `docker build`, `npm run build`, 린트 실제 실행
- 필수/경고 구분된 리포트 + 수정 가이드

---

## 관리자 가이드 (인프라팀)

### 플러그인 업데이트

1. 해당 플러그인의 `skills/<name>/SKILL.md` 수정
2. `.claude-plugin/plugin.json`에서 `version` bump (semver)
3. commit + push to main
4. 개발자들은 다음 Claude Code 세션에서 자동 업데이트

### 수동 업데이트 (긴급할 때)

개발자가 즉시 최신 버전을 받으려면:

```
/plugin marketplace update syntnote-lpad
```

### 새 플러그인 추가

1. 디렉토리 생성:
   ```
   <new-plugin>/
   ├── .claude-plugin/plugin.json
   └── skills/<skill-name>/SKILL.md
   ```
2. 루트의 `.claude-plugin/marketplace.json`의 `plugins` 배열에 항목 추가
3. commit + push

### 로컬 테스트

퍼블리시 전 검증:

```
/plugin marketplace add /Users/synt-ted/dev/lpad-infra/repos/lpad-devtools
```

```
/plugin install lpad-preflight@syntnote-lpad --scope user
```

## 구조

```
lpad-devtools/
├── .claude-plugin/
│   └── marketplace.json              # 마켓플레이스 카탈로그
└── lpad-preflight/
    ├── .claude-plugin/
    │   └── plugin.json               # 버전, 메타데이터
    └── skills/lpad-preflight/
        └── SKILL.md                  # 스킬 본체
```
