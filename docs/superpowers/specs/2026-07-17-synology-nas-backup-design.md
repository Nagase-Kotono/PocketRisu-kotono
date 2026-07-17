# 시놀로지 NAS 자동 백업 (하루 4번) — 설계

날짜: 2026-07-17
방식: **Approach B (보정판) — 앱은 신뢰 가능한 로컬 SSD에 깨끗한 풀 백업을 쓰고, 맥 launchd 스크립트가 하루 4번 트리거 + NAS로 복사**
앱 코드 변경: **없음** (server.cjs 안 건드림 → upstream merge conflict 위험 0)

## 목표

- 같은 LAN의 시놀로지 NAS(8TB)에 리수 데이터를 **하루 4번 자동 백업**.
- 리수에 큰 문제 생기면 **이전 백업본으로 복원(롤백)**.
- 오프-머신 사본이라 맥 외장 SSD가 통째로 죽어도 살아남게.

## 이미 있는 것 (재구현 금지)

백업/복원 인프라는 이미 거의 다 있음:

| 기능 | 위치 | 상태 |
|---|---|---|
| 자동 DB 스냅샷 (활동 기반, 5분 쿨다운, rotation 20/500MB, KV 내부 dedup) | `server.cjs:113~231` | DB만 |
| 풀 백업 저장 (asset·inlay·cold storage·DB 전부, tmp+rename atomic) | `POST /api/backup/server/save` (4091) | **수동만** |
| 풀 백업 복원 | `POST /api/backup/server/restore` (4247) | 있음 |
| 스냅샷 복원 (atomic, server-side) | `POST /api/db/snapshots/restore` (5440) | 있음 |
| 백업 경로 설정 (임의 절대경로, 앱파일 안쪽만 차단, 쓰기 probe) | `PUT /api/backup/server/path` (5530) | 있음, UI 있음 |
| 백업 파일 목록/다운로드/삭제 | `GET/DELETE /api/backup/server/*` (4218~4343) | 있음 |
| 백업 관리 UI | `SystemBackup.svelte`, `ServerBackupList.svelte` | 있음 |

**결론:** "복원"은 100% 존재. "자동 백업"도 DB 스냅샷은 사실상 자동.
**진짜 빠진 것:** ① 풀 백업(이미지 포함) 무인 스케줄, ② 풀 백업 파일 rotation, ③ NAS 오프사이트 복사.
→ 셋 다 **호스트 스크립트 + launchd + bind mount + config**로 해결. 앱 코드 불필요.

## 설계를 좌우한 사실들

1. **JWT는 5분짜리** (`createServerJwt`, `exp: now + 5*60`, `server.cjs:1496`). 시크릿 `save/__jwt_secret`, 비번 `save/__password`.
   → 스크립트가 `save/__password` 읽어 `POST /api/login` 하면 토큰. 하드코딩 없음. 매 실행 로그인(5분이면 충분).
2. **풀 백업 파일 rotation 없음.** `risu-backup-*.bin`은 저장만, 자동 삭제 안 함. → 스크립트가 retention.
3. **`checkActiveSession`은 `x-session-id` 없으면 통과** (`server.cjs:1450`). → 무인 호출 OK.
4. **Docker bind mount는 컨테이너 시작 시점에 고정.** SMB가 그때 안 붙어 있거나 중간에 끊기면 컨테이너가 빈/stale 폴더를 봄 → 백업이 조용히 실패. **← 이래서 컨테이너가 NAS에 직접 쓰지 않음.**

## 아키텍처: 2단 백업

```
컨테이너 ──(항상 붙은 외장 SSD)──> 로컬 백업 폴더 ──(스크립트 rsync)──> 시놀로지 NAS
         POST /save                /Volumes/Nagase_K/          /Volumes/risu-nas-backup
                                   poketrisu-backups           (SMB, 스크립트 실행 시에만 필요)
```

- **로컬 SSD 백업**: 앱이 직접, 항상 신뢰 가능. 빠른 복원(논리적 실수·DB 꼬임). UI 원클릭 복원 대상.
- **NAS 복사**: 스크립트가. 오프사이트 재해 복구(외장 SSD 사망). NAS 마운트는 스크립트 돌 때만 있으면 됨.
- NAS 마운트 죽으면? → 백업은 로컬에 안전, 복사만 다음 회차로 미뤄짐. **데이터 손실 0.**

## 구성

### 1) 컨테이너에 로컬 백업 폴더 bind (docker-compose.override.yml)
save bind mount 하는 override 파일(커밋 안 함)에 한 줄 추가:
```yaml
services:
  risuai:
    volumes:
      - /Volumes/Nagase_K/poketrisu:/app/save
      - /Volumes/Nagase_K/poketrisu-backups:/app/backups-data   # 추가 (외장 SSD, 항상 마운트됨)
```

### 2) 백업 경로 지정
설정 → Backups → 경로 변경에서 `/app/backups-data` 입력. (`PUT /api/backup/server/path`, 기존 UI)
→ 이후 풀 백업이 로컬 SSD로. **코드 0.**

### 3) SMB 마운트 (맥 → 시놀로지)
- 맥이 시놀로지 SMB 공유폴더를 고정 마운트포인트 `/Volumes/risu-nas-backup`에 마운트.
- Keychain에 SMB 자격증명 1회 저장(`open smb://...` 또는 `mount_smbfs`). → 스크립트가 무인 remount 가능.
- NAS 폴더에 sentinel 파일 `.risu-nas-alive` 1회 생성 (마운트 살았는지 증명용).

### 4) 호스트 스크립트 `backup-to-nas.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

SAVE_DIR="/Volumes/Nagase_K/poketrisu"
LOCAL_DIR="/Volumes/Nagase_K/poketrisu-backups"   # = 컨테이너 /app/backups-data
NAS_DIR="/Volumes/risu-nas-backup"                # SMB 마운트
SENTINEL="$NAS_DIR/.risu-nas-alive"
SMB_URL="//USER@SYNOLOGY-HOST/RisuBackup"         # remount용 (비번은 Keychain)
BASE="http://localhost:6001"
KEEP_LOCAL=30                                      # 최근 30개(4번×7.5일). 빠른 복원용, 작게
KEEP_NAS=120                                       # 최근 120개(4번×30일). 오프사이트 아카이브. 노브

# 1) 백업 트리거 → 로컬 SSD (항상 신뢰 가능)
PW="$(cat "$SAVE_DIR/__password")"
TOKEN="$(curl -sf -X POST "$BASE/api/login" -H 'content-type: application/json' \
  --data "$(jq -nc --arg p "$PW" '{password:$p}')" | jq -r '.token')"
[ -n "$TOKEN" ] && [ "$TOKEN" != "null" ] || { echo "login failed"; exit 1; }
LAST="$(curl -sf -N -X POST "$BASE/api/backup/server/save" -H "risu-auth: $TOKEN" | tail -n1)"
echo "$LAST" | grep -q '"type":"done"' || { echo "backup failed: $LAST"; exit 1; }

# 2) 로컬 retention
ls -1t "$LOCAL_DIR"/risu-backup-*.bin 2>/dev/null | tail -n +$((KEEP_LOCAL+1)) | xargs -r rm -f

# 3) NAS 마운트 살았나? 죽었으면 remount 1회 시도
[ -f "$SENTINEL" ] || mount_smbfs "$SMB_URL" "$NAS_DIR" 2>/dev/null || true
if [ ! -f "$SENTINEL" ]; then
  echo "NAS unavailable — 백업은 로컬에 안전, 오프사이트 복사만 스킵"; exit 0
fi

# 4) 새 백업만 NAS로 복사 (이미 있는 건 스킵)
rsync -a --ignore-existing "$LOCAL_DIR"/risu-backup-*.bin "$NAS_DIR"/

# 5) NAS retention
ls -1t "$NAS_DIR"/risu-backup-*.bin 2>/dev/null | tail -n +$((KEEP_NAS+1)) | xargs -r rm -f
```
- 의존성: `jq` (`brew install jq`). 비번에 특수문자 있어도 안전하게 JSON 조립.

### 5) launchd (하루 4번)
- `~/Library/LaunchAgents/com.kotono.risu-nas-backup.plist`
- `StartCalendarInterval`로 00·06·12·18시 실행. stdout/stderr는 로그 파일로.

## 복원 절차 (새로 만들 것 없음)

1. **흔한 경우 (최근 백업, 로컬에 있음):** 설정 → Backups → 서버 백업 목록에서 선택 → 복원 버튼. (`POST /api/backup/server/restore`)
2. **오래된 NAS-only 백업:** NAS의 `.bin`을 `/Volumes/Nagase_K/poketrisu-backups`에 복사 → UI 목록에 뜸 → 복원 버튼.
3. **외장 SSD 사망 재해:** 새 환경에 리수 띄우고 → NAS의 `.bin`을 백업 경로에 놓고 → 복원 버튼.

## 리스크 & 노브

| 리스크 | 대응 |
|---|---|
| **백업 경로 config 어긋남 → 조용한 no-op** (리뷰 지적 #1) | done JSON의 `filename`으로 로컬 착지 검증. 파일 없으면 fail + 알림 |
| **NAS 복사 전 로컬 retention이 미동기화분 삭제** (리뷰 지적 #2) | NAS 복사를 retention보다 먼저. 로컬은 NAS에 사본 있는 것만 삭제 |
| **NAS 장기 장애가 조용히 지속** (리뷰 지적 #3) | heartbeat(`.last-success`) 미갱신 + 매 회차 osascript 알림 |
| SMB 마운트 끊김/미마운트 | 컨테이너는 NAS 안 씀(로컬만). 스크립트가 `mount` 테이블+sentinel 둘 다 확인 후 remount 시도, 실패해도 로컬 백업은 안전 |
| NAS 장기 장애 중 로컬 SSD 증가 | 미동기화분을 안 지우므로 로컬이 늘어남. 알림 보고 NAS 복구가 정상 대응. (장애 수십 일 방치 시 로컬 용량 주의) |
| 백업 무한 적재 | 스크립트 retention (로컬 30, NAS 120). 노브 한 줄 |
| 풀 백업은 dedup 안 됨 (통짜) | KEEP × 전체용량으로 쌓임. 8TB라 여유. 용량 늘면 KEEP 조정 |
| JWT 5분 만료 | 매 실행 로그인 |
| 살아있는 SQLite 정합성 | 앱이 flushPendingDb 후 atomic tmp+rename로 조립 → 애초에 문제없음 |
| 비번 미설정 시 로그인 불가 | 전제: 비번 설정돼 있어야 함. 없으면 먼저 설정 |
| launchd가 슬립 중 못 돎 | 놓친 회차는 다음에 커버. 4번/일이라 공백 짧음 |

## 이중 리뷰 반영 (2026-07-17)

Codex + Claude 독립 리뷰 결과 **둘 다 "조건부 승인"**. 겹쳐 지적한 MUST-FIX 3개(로컬 착지 검증 / NAS-복사-우선·미동기화 보호 / 실패 가시성)를 스크립트에 반영함. 검증되어 안 건드리는 부분: 무인 인증(JWT+session 통과), atomic rename, done-grep 판정, PATH·launchd 설정. `xargs -r`는 이 머신(Darwin 27)에서 결함 아님이나 while-read 루프로 대체함.

**갈린 결정 2개 (확정됨, 2026-07-17):**
- **빈도 = 하루 4번** (사용자 확정). 리뷰어 둘 다 "1~2번" 권고했으나 8TB·오프사이트 이유로 4번 선택. idle 스킵 최적화는 SQLite라 보류.
- **동시 실행 lock = 없음** (사용자 확정, Claude 손). launchd 직렬화 + 고유 파일명(Date.now()) + atomic rename이라 수동+예약이 겹쳐도 손상 경로가 없음.

## 검증

- 스크립트 수동 1회 → 로컬에 `.bin` 생기고 `done` 뜨는지, NAS로 복사되는지.
- sentinel 지우고 실행 → 로컬 백업은 되고 NAS 복사만 스킵(exit 0)되는지.
- KEEP 넘겨 여러 개 → 로컬·NAS 각각 오래된 것 지워지는지.
- 설정 UI에서 그 백업으로 복원 → 데이터 돌아오는지.

## 스코프 밖 (YAGNI)

- 인앱 스케줄러(setInterval): launchd로 충분, 앱 코드 늘리면 upstream 충돌만.
- 백업 암호화·압축: `.bin`은 이미 자체 포맷. 필요해지면 그때.
- 클라우드 오프사이트: NAS 자체 Cloud Sync로 별도 처리.
