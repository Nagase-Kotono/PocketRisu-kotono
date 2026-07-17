# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

PocketRisu는 self-hosted AI roleplay chat 플랫폼이에요 (RisuAI 파생, GPL-3.0). Svelte 5 + TypeScript SPA 프론트엔드와 Node/Express 단일 서버로 구성되고, 모든 데이터는 서버의 SQLite 하나에 저장돼요.

이 레포는 **개인 포크**예요.

- upstream: `https://github.com/PocketRisu/PocketRisu.git` (원본)
- origin: `https://github.com/Nagase-Kotono/PocketRisu-kotono.git` (이 포크)

## 명령어

Node >= 22.12, pnpm 10.34.1 (packageManager 필드로 고정).

```bash
pnpm dev            # Vite dev 서버
pnpm build          # 프로덕션 빌드 (dist/)
pnpm check          # svelte-check 타입 검사
pnpm test           # vitest 전체 (기본 config)
pnpm test:compat    # RisuAI 호환성 테스트 (test/compat, vitest.config.compat.ts)
pnpm test:server    # 서버 테스트 (vitest.config.server.ts)
pnpm runserver      # Node 서버 실행 (빌드된 dist/ 서빙, 기본 포트 6001, PORT env로 변경)
```

단일 테스트 파일: `pnpm vitest run src/ts/util.test.ts`

서버를 실제로 띄우려면 `pnpm build` 후 `pnpm runserver` 순서로 실행해요 (서버가 `dist/`를 서빙하기 때문).

## 아키텍처

### 프론트엔드 (`src/`)

- 진입점 `src/main.ts` → `src/App.svelte`. Vite + Svelte 5 (runes) + Tailwind CSS 4.
- `src/ts/` — UI와 분리된 핵심 로직 계층:
  - `process/` — 채팅 요청 파이프라인 (프롬프트 조립, 요청 전송, 응답 처리)
  - `model/` — AI provider 정의 (OpenAI, Claude, Gemini, Ollama 등)
  - `storage/` — 저장 계층 추상화 (서버 SQLite / 로컬 저장 분기)
  - `parser/` — CBS(Curly Brace Syntax) 및 markdown 파싱
  - `plugins/`, `translator/`, `characters.ts`, `characterCards.ts` — 플러그인, 자동 번역, 캐릭터 카드 (`.charx`, `.risum`, `.risup`)
  - `stores.svelte.ts`, `globalApi.svelte.ts` — 전역 상태
- `src/lib/` — Svelte 컴포넌트 (`ChatScreens/`, `SideBars/`, `Setting/`, `Mobile/`, `UI/`)
- `src/lang/` — UI 다국어

### 서버 (`server/`)

- `server/node/server.cjs` — Express 단일 파일 서버. `db.cjs`가 better-sqlite3로 SQLite를 관리하고, 데이터는 실행 위치 기준 `save/` 폴더에 저장돼요 (백업, 비밀번호, JWT secret, inlay 이미지 포함).
- `server/hono/` — Cloudflare Workers용 대체 서버 (별도 package.json).

### 테스트

vitest config가 3개로 분리돼 있어요: 기본(`vitest.config.ts`, src 내 `*.test.ts`), compat(`test/compat`, RisuAI 데이터 호환성), server(`server/` 테스트).

### Docker

- 멀티스테이지 `Dockerfile`, 이미지 `ghcr.io/pocketrisu/pocketrisu`, 포트 6001, 데이터 볼륨 `/app/save`.
- `docker-compose.yml`의 프로젝트/컨테이너/볼륨 이름(`risuai-nodeonly`)은 v1.5.x 사용자 호환용이라 **변경하면 안 돼요** (파일 내 주석 참고).
- 이 머신에서 서버를 띄울 때 세이브 데이터는 named volume 대신 `/Volumes/Nagase_K/poketrisu` 바인드 마운트를 사용해요. compose 파일을 직접 고치지 말고 `docker-compose.override.yml`로 덮어써요 (override 파일은 커밋하지 않음):

```yaml
services:
  risuai:
    volumes:
      - /Volumes/Nagase_K/poketrisu:/app/save
```

- 이 머신의 Docker 런타임은 OrbStack이고, `data_dir`가 `/Volumes/Nagase_K/docker`로 설정돼 있어서 이미지·컨테이너·named volume이 전부 외장 SSD에 저장돼요. 맥미니 내장 디스크에는 Docker 데이터가 쌓이지 않아요.

## 포크 운영: upstream 동기화

원본(upstream)을 주기적으로 추적하고, 업데이트가 생기면 바로 이 포크에도 반영해요.

```bash
git fetch upstream
git checkout main && git merge upstream/main
git push origin main
```

- 동기화 merge는 fast-forward가 가능하면 fast-forward로 처리해요.
- 로컬 수정과 충돌이 나면 upstream 쪽 변경 의도를 먼저 파악하고 나서 해소해요.

## 버전 정책

- `package.json`의 `version`은 upstream 그대로 두고 **절대 수정하지 않아요**. 이 값은 클라이언트(`__APP_VERSION__`)와 서버 업데이트 체크(upstream update worker에 전송)에 쓰이기 때문에, suffix를 붙이면 upstream 릴리스마다 merge conflict가 나고 업데이트 알림이 깨질 수 있어요.
- 포크 자체 변경을 식별할 때는 git 태그를 써요: `v<upstream버전>-kotono.<N>` (예: `v1.8.1-kotono.1`). docker 빌드 구분이 필요하면 이미지 태그(예: `pocketrisu-kotono:1.8.1-k1`)를 써요.
- `version.json`은 코드 어디에서도 참조되지 않는 legacy 파일이에요. 건드리지 않아요.

## 브랜치 전략: Git Flow

- `main` — 릴리스 브랜치 (upstream 동기화 대상)
- `develop` — 통합 브랜치
- `feature/*` — 기능 작업 (develop에서 분기, develop으로 merge)
- `hotfix/*` — 긴급 수정 (main에서 분기)
- `release/*` — 릴리스 준비

## 이중 검증 프로토콜 (dual review)

non-trivial 작업은 확정 전에 독립 리뷰어 2개로 교차 검증해요. 대상은 계획 수립 후 평가, 구현·수정 완료 후 검증, 아키텍처 결정, 버그 근본 원인 수정이에요. 한 파일 오타·포맷 수정 같은 trivial 작업은 건너뛰어요.

리뷰어 오케스트레이션은 **Orca로만** 해요. `omc ask codex`·omc teams·Claude Code 내장 Agent tool은 쓰지 않아요 (사용자가 Orca 앱 UI에서 worker 터미널을 직접 봐야 하고, 오케스트레이션 경로를 Orca로 통일하기 때문).

1. leader가 Orca orchestration으로 **독립 리뷰어 2개**를 띄워요: `orca orchestration task-create --spec ...` → `orca terminal create --worktree active --command "claude" ...` → `orca orchestration dispatch --task <id> --to <handle> --inject` → `orca orchestration check --wait --types worker_done,escalation` (상세는 `orca skills get orchestration`).
2. 두 리뷰어는 진행 중인 세션 컨텍스트를 전혀 모르는 깨끗한 환경에서 critic 역할로 돌리고, 대화 맥락 없이 산출물(계획 문서, diff, 테스트 결과)만 전달해요. 서로 독립이어야 해요 (같은 산출물을 각자 따로 평가).

두 리뷰 결과를 leader가 종합해서 지적사항을 반영하고, 둘 다 통과했을 때만 계획 확정이나 완료 선언을 해요. 두 리뷰어의 의견이 충돌하면 leader가 임의로 결정하지 말고, 상반된 의견과 각각의 근거를 정리해서 사용자에게 보고하고 결정을 받아요.

## 커밋 메시지 컨벤션

```
[tag]: 한국어로 자세한 설명

- 세부 변경사항 1
- 세부 변경사항 2
```

- 제목 줄 뒤에 반드시 빈 줄 1줄을 두고, 그 다음 줄부터 `-`로 세부 항목을 나열해요. 빈 줄이 없으면 `git log --oneline`에서 제목과 본문이 한 줄로 합쳐져 보여요 (Git 표준: subject + blank line + body).
- tag는 대괄호로 감싸요 (예: `[fix]: ...`, `[chore]: ...`). scope가 필요하면 대괄호 안에 넣어요 (예: `[fix(chat)]: ...`). upstream은 대괄호 없는 표기를 쓰지만, 이 포크의 자체 커밋은 대괄호 표기를 써요.
- 설명은 한국어로 작성하고, 기술 용어와 코드 식별자는 영어 원문을 유지해요.

### tag — Conventional Commits v1.0.0 표준 11개만 사용 (새 타입 창조 금지)

| tag | 용도 | 이 프로젝트 적용 예 |
|---|---|---|
| `feat` | 새 기능 추가 | 새 UI 컴포넌트, AI provider 추가, 서버 엔드포인트 추가 |
| `fix` | 버그 수정 | 잘못된 상태 갱신, 파싱 오류, 서버 응답 오류 수정 |
| `docs` | 문서만 변경 (코드 변경 없음) | README, docs/, 코드 주석 |
| `style` | 코드 의미에 영향 없는 포맷 변경 | 들여쓰기, 공백, 임포트 정렬 |
| `refactor` | 기능·버그 변경 없이 구조 개선 | 컴포넌트 분리, 함수 추출, 변수명 정리 |
| `perf` | 성능 개선 | 불필요한 재렌더 제거, 쿼리·캐시 최적화 |
| `test` | 테스트 추가/수정 | vitest 테스트, compat fixture |
| `build` | 빌드 시스템·의존성 변경 | Dockerfile, package.json, vite.config.ts |
| `ci` | CI/CD 설정 변경 | .github/workflows |
| `chore` | 위 어디에도 안 들어가는 잡다한 변경 | .gitignore, 파일 이동, 버전 bump |
| `revert` | 이전 커밋 되돌리기 | `git revert`로 생성된 커밋 |

의무 타입은 `feat`/`fix` 2개(Semantic Versioning 자동 계산용)이고, 한 커밋이 여러 타입에 걸치면 가장 큰 변경의 타입을 고르거나 커밋을 나눠요.
