# lpad-claude-plugins

신트노트 lpad 개발자용 Claude Code 플러그인 마켓플레이스.

## 설치 (개발자용)

> **사전 조건:** 이 repo는 private이다. GitHub 인증 필요.
>
> `gh auth status`로 로그인 확인. 안 되어 있으면 `gh auth login`.
> (백그라운드 자동 업데이트를 원하면 `GITHUB_TOKEN` 환경변수도 설정)

Claude Code에서:

```bash
# 마켓플레이스 등록 (최초 1회)
/plugin marketplace add syntnote/lpad-claude-plugins

# 플러그인 설치
/plugin install lpad-preflight@syntnote-lpad
```

설치 후에는 자동 업데이트된다.

## 플러그인 목록

| 이름 | 용도 | 버전 |
|------|------|------|
| `lpad-preflight` | lpad 배포 전 프로젝트 사전 점검 | 0.1.0 |

### lpad-preflight

자기 프로젝트 디렉토리에서:

```
/lpad-preflight
```

- 트랙(Amplify/ECS) 자동 감지
- Dockerfile, .gitignore, 헬스체크 등 정적 검사
- `docker build`, `npm run build`, 린트 실제 실행
- 필수/경고 구분된 리포트 + 수정 가이드

---

## 관리자 가이드 (인프라팀)

### 플러그인 업데이트

1. 해당 플러그인의 `skills/<name>/SKILL.md` 수정
2. 해당 플러그인의 `.claude-plugin/plugin.json`에서 `version` bump (semver)
3. commit + push to main
4. 개발자들은 다음 Claude Code 세션에서 자동 업데이트

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

플러그인 퍼블리시 전 검증:

```bash
# Claude Code에서
/plugin marketplace add /Users/synt-ted/dev/lpad-infra/repos/lpad-claude-plugins
/plugin install lpad-preflight@syntnote-lpad
```

## 구조

```
lpad-claude-plugins/
├── .claude-plugin/
│   └── marketplace.json              # 마켓플레이스 카탈로그
└── lpad-preflight/
    ├── .claude-plugin/
    │   └── plugin.json               # 버전, 메타데이터
    └── skills/lpad-preflight/
        └── SKILL.md                  # 스킬 본체
```
