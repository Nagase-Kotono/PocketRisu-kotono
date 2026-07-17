# 시놀로지 NAS 자동 백업 (하루 4번) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 리수 풀 백업을 하루 4번 외장 SSD 로컬에 자동 생성하고 시놀로지 NAS로 복사해서, 문제 시 이전 백업으로 원클릭 복원한다.

**Architecture:** 앱 코드는 안 건드린다. 컨테이너는 항상 붙어있는 외장 SSD의 로컬 폴더에 기존 풀 백업 API(`POST /api/backup/server/save`)로 백업을 쓴다(신뢰 가능). 맥 launchd가 하루 4번 호스트 스크립트를 돌려 백업을 트리거하고, 결과 `.bin`을 시놀로지 SMB 마운트로 rsync 복사한 뒤 로컬·NAS 양쪽 retention을 돌린다. NAS 마운트가 죽어도 백업은 로컬에 남으므로 데이터 손실이 없다.

**Tech Stack:** bash, curl, jq, rsync, mount_smbfs, macOS launchd, Docker Compose(OrbStack), PocketRisu 기존 백업/복원 API·UI.

## Global Constraints

- 앱 코드(`server.cjs` 등) 변경 금지 — upstream merge conflict 회피. 전부 config·infra·호스트 스크립트로만 해결.
- `package.json`의 `version` 수정 금지 (CLAUDE.md 버전 정책).
- `docker-compose.yml` 직접 수정 금지 — 반드시 `docker-compose.override.yml`로 덮어씀. override는 커밋하지 않음 (CLAUDE.md).
- 스크립트·plist는 머신 특정 경로를 담으므로 레포에 커밋하지 않고 호스트에 둔다 (override와 동일 원칙).
- 머신 고정 경로: save = `/Volumes/Nagase_K/poketrisu`, Docker data_dir = `/Volumes/Nagase_K/docker`, 서버 포트 6001.
- 전제: 리수에 비밀번호가 설정돼 있어야 한다 (`save/__password` 존재). 없으면 설정 UI에서 먼저 지정.

---

## File Structure

| 파일 | 위치 | 책임 |
|---|---|---|
| docker-compose.override.yml | 레포 루트 (커밋 안 함) | 로컬 백업 폴더를 컨테이너에 bind |
| risu-backup-to-nas.sh | `~/.local/bin/` (호스트, 커밋 안 함) | 백업 트리거 + NAS 복사 + retention |
| com.kotono.risu-nas-backup.plist | `~/Library/LaunchAgents/` (호스트) | 하루 4번 스케줄 |
| 로컬 백업 폴더 | `/Volumes/Nagase_K/poketrisu-backups/` | 앱이 쓰는 로컬 백업 저장소 |
| NAS 백업 폴더 | 시놀로지 공유 `RisuBackup`, 맥에 `/Volumes/risu-nas-backup`로 마운트 | 오프사이트 아카이브 |
| 로그 | `~/Library/Logs/risu-nas-backup.log` | launchd stdout/stderr |

---

## Task 1: 로컬 백업 폴더 + 컨테이너 bind + 백업 경로 지정

앱이 신뢰 가능한 외장 SSD 로컬 폴더에 백업을 쓰도록 만든다. NAS는 아직 관여 안 함.

**Files:**
- Create: `/Volumes/Nagase_K/poketrisu-backups/` (디렉토리)
- Modify: `docker-compose.override.yml` (레포 루트, 커밋 안 함)

**Interfaces:**
- Produces: 백업 경로 `/app/backups-data`(컨테이너) = `/Volumes/Nagase_K/poketrisu-backups`(호스트). Task 3이 이 호스트 경로를 `LOCAL_DIR`로 읽는다.

- [ ] **Step 1: 로컬 백업 폴더 생성**

```bash
mkdir -p /Volumes/Nagase_K/poketrisu-backups
```

- [ ] **Step 2: override에 bind 추가**

`docker-compose.override.yml` (없으면 생성, 있으면 volumes에 줄 추가):

```yaml
services:
  risuai:
    volumes:
      - /Volumes/Nagase_K/poketrisu:/app/save
      - /Volumes/Nagase_K/poketrisu-backups:/app/backups-data
```

- [ ] **Step 3: 컨테이너 재생성**

```bash
cd /Volumes/Nagase_K/Kotono_github/PocketRisu-kotono/feature-docker-update
docker compose up -d --force-recreate
```

- [ ] **Step 4: bind 확인**

Run:
```bash
docker compose exec risuai sh -c 'touch /app/backups-data/.probe && ls -la /app/backups-data/.probe && rm /app/backups-data/.probe'
ls -la /Volumes/Nagase_K/poketrisu-backups/
```
Expected: `.probe`가 컨테이너에서 생성되고 호스트 `/Volumes/Nagase_K/poketrisu-backups/`에서도 보였다가 지워짐 (bind 양방향 확인).

- [ ] **Step 5: 백업 경로를 로컬 폴더로 지정**

설정 UI(설정 → Backups → 경로 변경)에서 `/app/backups-data` 입력, 또는 API로:

```bash
# 비번으로 토큰 받기
PW="$(cat /Volumes/Nagase_K/poketrisu/__password)"
TOKEN="$(curl -sf -X POST http://localhost:6001/api/login -H 'content-type: application/json' \
  --data "$(jq -nc --arg p "$PW" '{password:$p}')" | jq -r '.token')"
# 경로 지정
curl -sf -X PUT http://localhost:6001/api/backup/server/path \
  -H "risu-auth: $TOKEN" -H 'content-type: application/json' \
  --data '{"path":"/app/backups-data"}'
```
Expected: JSON에 `"path":"/app/backups-data"` 포함.

- [ ] **Step 6: 수동 풀 백업 1회 → 로컬에 떨어지는지 확인**

Run:
```bash
curl -sf -N -X POST http://localhost:6001/api/backup/server/save -H "risu-auth: $TOKEN" | tail -n1
ls -lh /Volumes/Nagase_K/poketrisu-backups/risu-backup-*.bin
```
Expected: 마지막 줄이 `{"type":"done","ok":true,"filename":"risu-backup-<ts>.bin",...}`, 호스트 폴더에 그 `.bin` 파일 존재.

> 커밋 없음: override는 커밋하지 않는다 (Global Constraints).

---

## Task 2: SMB 마운트 + Keychain 자격증명 + sentinel

맥이 시놀로지 공유를 고정 지점에 마운트하고, 스크립트가 무인 remount할 수 있게 자격증명을 Keychain에 넣고, 마운트 살아있음 판별용 sentinel을 만든다.

**Files:**
- Create: 시놀로지 공유폴더 `RisuBackup` (DSM에서), sentinel `.risu-nas-alive`
- Mount point: `/Volumes/risu-nas-backup`

**Interfaces:**
- Produces: `/Volumes/risu-nas-backup`(마운트포인트), `/Volumes/risu-nas-backup/.risu-nas-alive`(sentinel), SMB URL `//<user>@<synology-host>/RisuBackup`. Task 3이 `NAS_DIR`, `SENTINEL`, `SMB_URL`로 읽는다.

- [ ] **Step 1: 시놀로지에 공유폴더 + 백업 전용 계정 준비**

DSM → 제어판 → 공유폴더 → `RisuBackup` 생성. 백업 전용 계정에 이 폴더 읽기/쓰기 권한 부여. SMB 서비스 활성 확인 (제어판 → 파일 서비스 → SMB).

- [ ] **Step 2: 마운트포인트 생성 + 최초 마운트 (Keychain 저장)**

```bash
mkdir -p /Volumes/risu-nas-backup
# Finder로 open smb://<synology-host>/RisuBackup 후 "키체인에 암호 저장" 체크가 가장 확실.
# 또는 CLI (암호 입력하면 Keychain에 저장됨):
mount_smbfs //<user>@<synology-host>/RisuBackup /Volumes/risu-nas-backup
```
Expected: `/Volumes/risu-nas-backup`에 공유폴더 내용이 보임. `mount | grep risu-nas-backup`에 smbfs 라인.

- [ ] **Step 3: sentinel 생성**

```bash
touch /Volumes/risu-nas-backup/.risu-nas-alive
ls -la /Volumes/risu-nas-backup/.risu-nas-alive
```
Expected: sentinel 파일 존재.

- [ ] **Step 4: 무인 remount 검증 (Keychain 자격증명이 실제로 먹히는지)**

Run:
```bash
umount /Volumes/risu-nas-backup 2>/dev/null || true
mount_smbfs //<user>@<synology-host>/RisuBackup /Volumes/risu-nas-backup   # 암호 프롬프트 없이 붙어야 함
ls /Volumes/risu-nas-backup/.risu-nas-alive
```
Expected: **암호 프롬프트 없이** 마운트되고 sentinel 보임. 프롬프트가 뜨면 Keychain 저장이 안 된 것 → Step 2를 Finder 방식으로 다시.

> 커밋 없음.

---

## Task 3: backup-to-nas.sh 스크립트

백업 트리거 → 로컬 retention → NAS 마운트 확인/remount → NAS 복사 → NAS retention. launchd 최소 환경에서도 돌게 PATH를 스크립트에서 고정한다.

**Files:**
- Create: `~/.local/bin/risu-backup-to-nas.sh`

**Interfaces:**
- Consumes: Task 1의 `LOCAL_DIR`(`/Volumes/Nagase_K/poketrisu-backups`), Task 2의 `NAS_DIR`·`SENTINEL`·`SMB_URL`, `save/__password`.
- Produces: `~/Library/Logs/risu-nas-backup.log`로 나가는 stdout/stderr(launchd가 리다이렉트), NAS의 `risu-backup-*.bin`. Task 4가 이 스크립트를 실행한다.

- [ ] **Step 1: 스크립트 작성**

`~/.local/bin/risu-backup-to-nas.sh` (`<user>`·`<synology-host>`는 실제 값으로 치환):

```bash
#!/usr/bin/env bash
# -e 제외: 빈 글롭+pipefail이 정상 흐름을 죽이는 걸 피하고, 필수 단계는 || fail로 명시 처리 (리뷰 지적)
set -uo pipefail
# launchd 최소 환경 대비: homebrew(jq) + system(rsync,curl,mount_smbfs) 경로 고정
export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/sbin:/usr/sbin"

SAVE_DIR="/Volumes/Nagase_K/poketrisu"
LOCAL_DIR="/Volumes/Nagase_K/poketrisu-backups"    # = 컨테이너 /app/backups-data
NAS_DIR="/Volumes/risu-nas-backup"
SENTINEL="$NAS_DIR/.risu-nas-alive"
SMB_URL="//<user>@<synology-host>/RisuBackup"      # 비번은 Keychain
BASE="http://localhost:6001"
KEEP_LOCAL=30                                       # 4번×7.5일. NAS 사본 있는 것만 삭제(미동기화분 보호)
KEEP_NAS=120                                        # 4번×30일 (오프사이트 아카이브). 노브
HEARTBEAT="$LOCAL_DIR/.last-success"                # 마지막 '오프사이트까지' 성공 시각(epoch)

ts() { date '+%Y-%m-%d %H:%M:%S'; }
notify() { osascript -e "display notification \"$1\" with title \"risu-backup\"" 2>/dev/null || true; }
fail() { echo "[$(ts)] FAIL: $*"; notify "리수 백업 실패 (로그 확인)"; exit 1; }
echo "[$(ts)] backup start"

# 1) 로그인 → 토큰 (실패 진단이 죽지 않게 curl 종료상태/본문 분리)
PW="$(cat "$SAVE_DIR/__password" 2>/dev/null)" || fail "비번 파일 없음: $SAVE_DIR/__password"
LOGIN="$(curl -sf --max-time 30 -X POST "$BASE/api/login" -H 'content-type: application/json' \
  --data "$(jq -nc --arg p "$PW" '{password:$p}')")" || fail "login 요청 실패 (서버 다운?)"
TOKEN="$(printf '%s' "$LOGIN" | jq -r '.token // empty')"
[ -n "$TOKEN" ] || fail "토큰 없음 (비번 틀림/미설정?)"

# 2) 백업 트리거 → 로컬 SSD
LAST="$(curl -sf -N --max-time 1800 -X POST "$BASE/api/backup/server/save" -H "risu-auth: $TOKEN" | tail -n1)" \
  || fail "save 요청 실패"
printf '%s' "$LAST" | jq -e '.type=="done" and .ok==true and (.filename|length>0)' >/dev/null 2>&1 \
  || fail "백업 미완료: $LAST"
FN="$(printf '%s' "$LAST" | jq -r '.filename')"

# 3) 로컬 착지 검증 — 리뷰 합의 MUST-FIX #1: done만 믿으면 backupsDir 어긋났을 때 조용한 no-op
[ -f "$LOCAL_DIR/$FN" ] || fail "API는 done인데 $FN 이 $LOCAL_DIR 에 없음 (백업 경로 config 어긋남?)"
echo "[$(ts)] local backup ok: $FN"

# 4) NAS 마운트 실확인 — sentinel 파일만이 아니라 mount 테이블도 확인(stale/이중마운트 방어). 죽었으면 remount 1회
nas_live() { mount | grep -q " on $NAS_DIR " && [ -f "$SENTINEL" ]; }
nas_live || mount_smbfs "$SMB_URL" "$NAS_DIR" 2>/dev/null || true

if nas_live; then
  # 5) NAS 복사 먼저 (retention보다 앞!). rsync 기본 비교로 손상/부분 파일도 재동기화
  rsync -rt --no-owner --no-group --no-perms "$LOCAL_DIR"/risu-backup-*.bin "$NAS_DIR"/ \
    || fail "rsync NAS 복사 실패"
  echo "[$(ts)] synced to NAS"
  # 6) NAS retention
  ls -1t "$NAS_DIR"/risu-backup-*.bin 2>/dev/null | tail -n +$((KEEP_NAS+1)) | while read -r f; do rm -f "$f"; done
  # 7) 로컬 retention — 리뷰 MUST-FIX #2: NAS에 사본이 있는 것만 삭제(미동기화분 절대 안 지움)
  ls -1t "$LOCAL_DIR"/risu-backup-*.bin 2>/dev/null | tail -n +$((KEEP_LOCAL+1)) | while read -r f; do
    [ -f "$NAS_DIR/$(basename "$f")" ] && rm -f "$f"
  done
  date +%s > "$HEARTBEAT"
  echo "[$(ts)] done (NAS synced)"
else
  # 리뷰 MUST-FIX #3: 조용히 넘기지 않음. 로컬 백업은 성공(exit 0)하되 heartbeat 미갱신 + 알림.
  # 로컬 retention도 스킵 → 미동기화 백업 보존(장기 장애 시 로컬이 늘어남, 아래 리스크표 참고).
  echo "[$(ts)] WARN: NAS 사용불가 — 로컬 백업만 완료, 오프사이트 복사 스킵"
  notify "NAS 오프사이트 백업 스킵 — 마운트 확인 필요"
fi
```

- [ ] **Step 2: 의존성 확인 + 실행권한**

Run:
```bash
command -v jq || brew install jq
command -v rsync
chmod +x ~/.local/bin/risu-backup-to-nas.sh
```
Expected: jq·rsync 경로 출력, chmod 성공.

- [ ] **Step 3: 정상 경로 실행 (test: 로컬+NAS 둘 다 생성)**

Run:
```bash
~/.local/bin/risu-backup-to-nas.sh
ls -lh /Volumes/Nagase_K/poketrisu-backups/risu-backup-*.bin | tail -1
ls -lh /Volumes/risu-nas-backup/risu-backup-*.bin | tail -1
```
Expected: 로그에 `local backup ok` → `synced to NAS` → `done`. 로컬·NAS 양쪽에 동일 `.bin` 존재.

- [ ] **Step 4: 마운트 끊김 시나리오 (test: 로컬은 되고 NAS만 스킵)**

> ⚠️ 실제 NAS의 sentinel을 지우지 않게, **umount 성공을 확인한 뒤에만** 진행한다 (리뷰 지적). sentinel은 NAS에 있으므로 언마운트되면 자동으로 안 보인다 — 따로 rm 하지 않는다.

Run:
```bash
umount /Volumes/risu-nas-backup || { echo "umount 실패(사용 중) — 테스트 중단"; }
mount | grep -q " on /Volumes/risu-nas-backup " && echo "아직 마운트됨 — 테스트 중단" || \
  { ~/.local/bin/risu-backup-to-nas.sh; echo "exit=$?"; }
cat /Volumes/Nagase_K/poketrisu-backups/.last-success 2>/dev/null; echo " <- heartbeat(미갱신이어야 함)"
```
Expected: 로그에 `WARN: NAS 사용불가 ... 스킵`, **exit=0**, 로컬 폴더엔 새 `.bin` 생성됨, `.last-success`(heartbeat)는 **갱신 안 됨**, 알림 뜸. (remount가 Keychain으로 다시 붙었다면 스킵 대신 정상 복사 — 그것도 OK.)

- [ ] **Step 5: retention (test: 오래된 것 삭제, 미동기화분 보호)**

Run:
```bash
# (a) 정상: 로컬은 NAS 사본 있는 것만, NAS는 오래된 것 삭제 — 개수 확인
ls -1 /Volumes/Nagase_K/poketrisu-backups/risu-backup-*.bin | wc -l   # KEEP_LOCAL 이하
ls -1 /Volumes/risu-nas-backup/risu-backup-*.bin | wc -l              # KEEP_NAS 이하
# (b) 미동기화분 보호: NAS에 없는 로컬 파일은 KEEP_LOCAL 넘겨도 안 지워지는지
#     (NAS 끊긴 상태로 여러 번 돌린 뒤 로컬 개수가 계속 늘어나는지 = 정상)
```
Expected: (a) 로컬 ≤ KEEP_LOCAL, NAS ≤ KEEP_NAS로 수렴. (b) NAS 장애 중에는 로컬이 KEEP_LOCAL을 넘겨 늘어남(미동기화 백업을 안 지우므로) — 이게 의도된 동작.

> 커밋 없음: 스크립트는 머신 특정 경로/호스트를 담아 호스트에만 둔다.

---

## Task 4: launchd 하루 4번 스케줄

00·06·12·18시에 스크립트를 자동 실행하고 로그를 남긴다.

**Files:**
- Create: `~/Library/LaunchAgents/com.kotono.risu-nas-backup.plist`

**Interfaces:**
- Consumes: Task 3의 `~/.local/bin/risu-backup-to-nas.sh`.

- [ ] **Step 1: plist 작성**

`~/Library/LaunchAgents/com.kotono.risu-nas-backup.plist` (`<user>`는 실제 홈, 예 `/Users/kotono`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.kotono.risu-nas-backup</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/kotono/.local/bin/risu-backup-to-nas.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Hour</key><integer>0</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>6</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>18</integer><key>Minute</key><integer>0</integer></dict>
  </array>
  <key>RunAtLoad</key>
  <false/>
  <key>StandardOutPath</key>
  <string>/Users/kotono/Library/Logs/risu-nas-backup.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/kotono/Library/Logs/risu-nas-backup.log</string>
</dict>
</plist>
```

- [ ] **Step 2: 로드**

```bash
launchctl unload ~/Library/LaunchAgents/com.kotono.risu-nas-backup.plist 2>/dev/null || true
launchctl load ~/Library/LaunchAgents/com.kotono.risu-nas-backup.plist
launchctl list | grep risu-nas-backup
```
Expected: `launchctl list`에 `com.kotono.risu-nas-backup` 라인 (PID는 대기 중이면 `-`).

- [ ] **Step 3: 즉시 1회 강제 실행 (test: 스케줄 배선 확인)**

Run:
```bash
launchctl kickstart -k gui/$(id -u)/com.kotono.risu-nas-backup
sleep 20
tail -n 20 ~/Library/Logs/risu-nas-backup.log
```
Expected: 로그에 이번 실행의 `backup start … done`(또는 NAS 스킵) 라인. launchd 경로로도 스크립트가 정상 동작.

- [ ] **Step 4: 스케줄 확정 확인**

Run:
```bash
launchctl print gui/$(id -u)/com.kotono.risu-nas-backup | grep -A6 'com.apple.launchd.calendarinterval\|next fire\|periodic'
```
Expected: 4개 캘린더 인터벌(0/6/12/18시)이 등록돼 있음. (출력 포맷은 macOS 버전마다 다르니 4개 시간대가 보이면 OK.)

> 커밋 없음.

---

## Self-Review

**Spec coverage:**
- 하루 4번 자동 백업 → Task 4 (launchd 0/6/12/18). ✅
- 로컬 SSD에 앱이 직접 백업 → Task 1. ✅
- NAS 오프사이트 복사 → Task 3 Step 4. ✅
- rotation(로컬30/NAS120) → Task 3 Step 2·5, 검증 Step 5. ✅
- 마운트 끊김 안전(데이터 손실 0) → Task 3 Step 4 검증. ✅
- 무인 인증(JWT 5분, 매 실행 로그인) → Task 3 Step 1. ✅
- 복원은 기존 UI/엔드포인트 → 스펙 "복원 절차", 앱 미변경. ✅
- 앱 코드 0 → 전 태스크 config·infra·호스트 스크립트만. ✅

**Placeholder scan:** `<user>`·`<synology-host>`는 머신 고유값이라 의도적 치환 지점 (플랜 본문에 "실제 값으로 치환" 명시). 나머지 경로·명령·코드는 전부 구체값. TODO/TBD 없음.

**Type consistency:** `LOCAL_DIR`/`NAS_DIR`/`SENTINEL`/`SMB_URL`/`KEEP_LOCAL`/`KEEP_NAS` 이름이 스크립트·Interfaces·검증에서 동일. 백업 경로 `/app/backups-data`(컨테이너) ↔ `/Volumes/Nagase_K/poketrisu-backups`(호스트) 매핑 Task 1·3 일치.

## 남은 실환경 값 (구현 시 채움)

- `<synology-host>`: 시놀로지 호스트명/IP.
- `<user>`: SMB 접속 계정.
- 홈 경로가 `/Users/kotono`가 아니면 plist·스크립트 경로 조정.
