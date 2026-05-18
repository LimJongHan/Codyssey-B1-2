# 과제 수행 튜토리얼

> macOS 환경 기준. `agent-app-leak`은 Linux 전용 바이너리이므로 Docker 컨테이너 안에서 실행해야 한다.

---

## 목차

1. [환경 준비 (Docker)](#1-환경-준비-docker)
2. [앱 실행 환경 구성](#2-앱-실행-환경-구성)
3. [monitor.sh 준비](#3-monitorsh-준비)
4. [케이스 1 - OOM (메모리 누수)](#4-케이스-1---oom-메모리-누수)
5. [케이스 2 - CPU Spike (과점유)](#5-케이스-2---cpu-spike-과점유)
6. [케이스 3 - Deadlock (교착상태)](#6-케이스-3---deadlock-교착상태)
7. [리포트 작성 가이드](#7-리포트-작성-가이드)

---

## 1. 환경 준비 (Docker)

### Docker 설치 확인

```bash
docker --version
# Docker Desktop이 없다면: https://www.docker.com/products/docker-desktop/
```

### Linux 컨테이너 실행

```bash
# B1-2 디렉터리를 컨테이너 안에 마운트해서 실행
docker run -it \
  --name codyssey-b12 \
  -v "$(pwd):/workspace" \
  ubuntu:22.04 \
  /bin/bash
```

> 이후 모든 명령어는 **컨테이너 내부 쉘**에서 실행한다.

### 컨테이너 안 기본 도구 설치

```bash
apt-get update && apt-get install -y \
  procps \
  htop \
  psmisc \
  curl \
  nano
```

- `procps`: `ps`, `top`, `kill` 등 포함
- `psmisc`: `pstree`, `killall` 포함

### 작업 디렉터리 이동

```bash
cd /workspace
ls -la
# agent-app-leak, Misson.md, study.md 등이 보여야 한다
```

---

## 2. 앱 실행 환경 구성

### 일반 사용자 생성

`agent-leak-app`은 root로 실행하면 실패한다. 일반 사용자로 실행해야 한다.

```bash
# 사용자 생성 (root 쉘에서 실행)
useradd -m -s /bin/bash agent-user
su - agent-user
# 이제 agent-user 쉘에서 작업한다
```

### 디렉터리 구조 생성

```bash
# AGENT_HOME 기준 디렉터리 생성
mkdir -p ~/agent/upload_files
mkdir -p ~/agent/api_keys
mkdir -p ~/agent/logs
```

### secret.key 파일 생성

```bash
echo "agent_api_key_test" > ~/agent/api_keys/secret.key
cat ~/agent/api_keys/secret.key   # 확인: agent_api_key_test 출력되어야 함
```

### 환경변수 설정 (.bash_profile)

```bash
nano ~/.bash_profile
```

아래 내용을 붙여넣는다.

```bash
export AGENT_HOME=/home/agent-user/agent
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=$AGENT_HOME/logs
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=80
export MULTI_THREAD_ENABLE=true
```

저장 후 적용:

```bash
source ~/.bash_profile
```

### 바이너리 실행 권한 부여

```bash
chmod +x /workspace/agent-app-leak
```

### 정상 실행 확인

```bash
/workspace/agent-app-leak
```

부트 시퀀스가 통과하면 로그가 출력되기 시작한다.
실패 시 어떤 조건이 빠졌는지 로그에 표시된다.

**종료**: `Ctrl + C`

---

## 3. monitor.sh 준비

`monitor.sh`는 별도 터미널(또는 백그라운드)에서 돌리면서 관제 데이터를 수집한다.

### 스크립트 작성

```bash
nano /workspace/monitor.sh
```

아래 내용을 붙여넣는다.

```bash
#!/bin/bash

LOG_FILE="/workspace/monitor.log"
INTERVAL=10   # 10초마다 기록

echo "=== Monitor Started: $(date) ===" >> "$LOG_FILE"

while true; do
    TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
    PROC_NAME="agent-app-leak"

    # 프로세스 정보 수집
    PROC_INFO=$(ps -eo pid,comm,%cpu,%mem --no-headers 2>/dev/null \
                | grep "$PROC_NAME" | head -1)

    if [ -z "$PROC_INFO" ]; then
        echo "[$TIMESTAMP] PROCESS:NOT_FOUND" >> "$LOG_FILE"
    else
        PID=$(echo $PROC_INFO | awk '{print $1}')
        CPU=$(echo $PROC_INFO | awk '{print $3}')
        MEM=$(echo $PROC_INFO | awk '{print $4}')
        DISK=$(df -h / | awk 'NR==2{print $4}')
        FW="active"   # 방화벽 상태 (환경에 따라 수정)

        echo "[$TIMESTAMP] PROCESS:$PROC_NAME CPU:${CPU}% MEM:${MEM}% DISK:$DISK FIREWALL:$FW" \
            >> "$LOG_FILE"
    fi

    sleep "$INTERVAL"
done
```

```bash
chmod +x /workspace/monitor.sh
```

### 사용법

터미널 1 (앱 실행):

```bash
/workspace/agent-app-leak 2>&1 | tee /workspace/app.log
```

터미널 2 (관제):

```bash
/workspace/monitor.sh &
tail -f /workspace/monitor.log
```

> **컨테이너에서 터미널을 2개 열려면**: 호스트(macOS)에서 새 터미널을 열고
> `docker exec -it codyssey-b12 su - agent-user` 를 실행한다.

---

## 4. 케이스 1 - OOM (메모리 누수)

### 목표

메모리가 시간 흐름에 따라 선형 증가하다 `MEMORY_LIMIT`에 도달하면 프로세스가 스스로 종료됨을 관측한다.

### Step 1: 낮은 MEMORY_LIMIT으로 실행 (Before)

```bash
export MEMORY_LIMIT=100   # 낮게 설정해서 빨리 재현
source ~/.bash_profile

# 터미널 1: 앱 실행 (로그 저장)
/workspace/agent-app-leak 2>&1 | tee /workspace/app_oom_before.log
```

```bash
# 터미널 2: 관제 시작
/workspace/monitor.sh &
tail -f /workspace/monitor.log
```

### Step 2: 관찰 포인트

monitor.log에서 MEM%가 점점 올라가는 것을 확인한다.

```bash
grep "MEM:" /workspace/monitor.log
```

app.log에서 종료 직전 로그를 확인한다.

```bash
grep -i "memory\|TERMINATED\|MemoryGuard" /workspace/app_oom_before.log
```

예상 출력:

```text
[CRITICAL] [MemoryGuard] Memory limit exceeded (100MB >= 100MB)
[CRITICAL] [MemoryGuard] Self-terminating process 1234 to prevent system instability.
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
```

### Step 3: MEMORY_LIMIT 조정 후 재실행 (After)

```bash
export MEMORY_LIMIT=512
source ~/.bash_profile

/workspace/agent-app-leak 2>&1 | tee /workspace/app_oom_after.log
```

- Before: 종료까지 걸린 시간 기록
- After: 더 오래 생존하는지 확인 후 `Ctrl+C`로 수동 종료

### Step 4: 증거 스크린샷 수집

```bash
# monitor.log에서 MEM 수치 추출
grep "MEM:" /workspace/monitor.log

# top으로 실시간 확인 (별도 터미널)
top -p $(pgrep agent-app-leak)
```

---

## 5. 케이스 2 - CPU Spike (과점유)

### 목표

CPU 사용률이 급격히 상승하다 `CPU_MAX_OCCUPY` 임계치를 초과하면 Watchdog이 SIGTERM을 보내 종료됨을 관측한다.

### Step 1: 낮은 CPU_MAX_OCCUPY로 실행 (Before)

```bash
export CPU_MAX_OCCUPY=10   # 낮게 설정해서 빨리 재현
export MEMORY_LIMIT=512    # 메모리는 충분히
source ~/.bash_profile

/workspace/agent-app-leak 2>&1 | tee /workspace/app_cpu_before.log
```

### Step 2: 관찰 포인트

monitor.log에서 CPU% 급상승 구간을 확인한다.

```bash
grep "CPU:" /workspace/monitor.log
```

app.log에서 Watchdog 로그를 확인한다.

```bash
grep -i "watchdog\|SIGTERM\|ABORT\|cpu" /workspace/app_cpu_before.log
```

예상 출력:

```text
[WARNING] [Watchdog] CPU usage exceeded limit (XX% >= 10%)
[SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)
```

top으로 CPU 급상승을 실시간 관찰:

```bash
# 별도 터미널에서
top -p $(pgrep agent-app-leak)
# 또는 1초 갱신으로
top -d 1 -p $(pgrep agent-app-leak)
```

### Step 3: CPU_MAX_OCCUPY 조정 후 재실행 (After)

```bash
export CPU_MAX_OCCUPY=90
source ~/.bash_profile

/workspace/agent-app-leak 2>&1 | tee /workspace/app_cpu_after.log
```

- Before: 종료 시각 및 CPU 수치 기록
- After: 종료되지 않거나 더 오래 생존하는지 비교

### Step 4: ps로 CPU 점유율 확인

```bash
# 특정 프로세스의 CPU 기록 (1초마다)
while true; do
    ps -p $(pgrep agent-app-leak) -o pid,%cpu,%mem --no-headers 2>/dev/null
    sleep 1
done
```

---

## 6. 케이스 3 - Deadlock (교착상태)

### 목표

멀티스레드 모드에서 스레드들이 서로의 자원을 무한히 기다리며 프로세스가 무응답 상태에 빠지는 것을 확인한다.

### Step 1: 멀티스레드 모드로 실행 (Deadlock 재현)

```bash
export MULTI_THREAD_ENABLE=true
export MEMORY_LIMIT=512
export CPU_MAX_OCCUPY=90
source ~/.bash_profile

/workspace/agent-app-leak 2>&1 | tee /workspace/app_deadlock.log
```

### Step 2: Deadlock 증상 확인

로그가 특정 시점 이후로 **완전히 멈추는** 것을 확인한다.

```bash
# 새 로그가 나오는지 실시간 관찰
tail -f /workspace/app_deadlock.log
# → 로그 출력이 멈추면 Deadlock 의심
```

**3가지 증거를 수집해야 한다.**

**증거 1: PID 존재 확인** (프로세스는 살아있음)

```bash
ps -ef | grep agent-app-leak
# PID가 출력되면 살아있는 것
```

**증거 2: CPU/MEM 변화 없음** (무응답 상태)

```bash
# 스레드별 CPU 사용률 확인
top -H -p $(pgrep agent-app-leak)
# 모든 스레드의 CPU가 0%이면 Deadlock

# 또는 ps로 스레드 상태 확인
ps -L -p $(pgrep agent-app-leak) -o tid,stat,%cpu,%mem
# STAT 컬럼이 모두 S(sleep) 또는 D(waiting)이면 Deadlock
```

**증거 3: 마지막 로그 확인**

```bash
tail -30 /workspace/app_deadlock.log
# WAITING, BLOCKED, LOCK 관련 로그가 마지막에 있는지 확인

grep -i "waiting\|blocked\|lock" /workspace/app_deadlock.log | tail -10
```

### Step 3: Deadlock 강제 종료

```bash
kill -9 $(pgrep agent-app-leak)
```

### Step 4: 싱글스레드 모드로 비교 (After)

```bash
export MULTI_THREAD_ENABLE=false
source ~/.bash_profile

/workspace/agent-app-leak 2>&1 | tee /workspace/app_nodeadlock.log
```

- After: 로그가 계속 출력되고 무응답 상태가 발생하지 않는 것을 확인

### Step 5: pstree로 스레드 구조 시각화

```bash
pstree -p $(pgrep agent-app-leak)
# 스레드 목록이 보임
```

---

## 7. 리포트 작성 가이드

### 디렉터리 생성

```bash
mkdir -p /workspace/reports
```

### 리포트 파일 구성

```bash
# 3개 파일 생성
touch /workspace/reports/oom.md
touch /workspace/reports/cpu.md
touch /workspace/reports/deadlock.md
```

### 리포트 템플릿

각 파일에 아래 구조로 작성한다.

```markdown
# [Bug] {장애 유형} - {한 줄 요약}

## 1. Description (현상 설명)
- 어떤 현상이 발생했는가?
- 언제, 어떤 조건(환경변수 설정)에서 발생했는가?

## 2. Evidence & Logs (증거 자료)

### monitor.log 발췌
(MEM% 또는 CPU% 변화 수치를 표 또는 코드블록으로)

### 프로그램 실행 로그 발췌
(종료 직전 핵심 로그 5~10줄)

### 시스템 도구 출력
(ps, top, pstree 출력 결과)

## 3. Root Cause Analysis (원인 분석)
- 수집된 증거를 근거로 기술적 원인 서술
- 관련 OS 동작 원리 설명 (MemoryGuard, Watchdog, 교착상태 조건 등)

## 4. Workaround & Verification (조치 및 검증)
- 어떤 환경변수를 어떻게 바꿨는가
- Before & After 비교 표
```

### 케이스별 Before & After 비교 표 예시

**OOM**

| 항목 | Before (MEMORY_LIMIT=100) | After (MEMORY_LIMIT=512) |
| --- | --- | --- |
| 생존 시간 | 약 X분 | 약 Y분 이상 |
| 최대 MEM% | 96.8% | 측정 중 |
| 종료 방식 | SELF-TERMINATED | 정상 동작 |

**CPU Spike**

| 항목 | Before (CPU_MAX_OCCUPY=10) | After (CPU_MAX_OCCUPY=90) |
| --- | --- | --- |
| 종료까지 시간 | 약 X분 | 미종료 또는 Y분 이상 |
| 최대 CPU% | XX% | YY% |
| 종료 방식 | SIGTERM (Watchdog) | 정상 동작 |

**Deadlock**

| 항목 | Before (MULTI_THREAD_ENABLE=true) | After (MULTI_THREAD_ENABLE=false) |
| --- | --- | --- |
| 프로세스 상태 | 무응답 (PID 존재) | 정상 동작 |
| 로그 출력 | 멈춤 | 계속 출력 |
| CPU% | 0% | 정상 수치 |

---

## 자주 하는 실수 & 해결법

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| 부트 시퀀스 실패 | 환경변수 미설정 또는 디렉터리 없음 | `env \| grep AGENT` 로 확인, 디렉터리 재생성 |
| root로 실행 시 실패 | 보안 정책 | `su - agent-user` 후 실행 |
| secret.key 오류 | 파일 내용 불일치 | `cat $AGENT_KEY_PATH/secret.key` 확인 |
| monitor.sh 에서 NOT_FOUND | 프로세스명 불일치 | `ps -ef` 로 실제 프로세스명 확인 후 수정 |
| 컨테이너 종료 후 재접속 | 컨테이너 중지됨 | `docker start codyssey-b12 && docker exec -it codyssey-b12 su - agent-user` |
| Deadlock인지 OOM인지 구분 불가 | 로그 미확인 | PID 존재하면 Deadlock, PID 없으면 OOM/CPU |

---

## 전체 수행 순서 요약

```text
1. Docker 컨테이너 실행 및 도구 설치
2. agent-user 생성 → 디렉터리 구조 생성 → secret.key 작성
3. .bash_profile 환경변수 설정 → source 적용
4. 바이너리 실행 권한 부여 → 정상 부팅 확인
5. monitor.sh 작성 → 백그라운드 실행

[케이스 1: OOM]
6. MEMORY_LIMIT=100 → 실행 → 종료 관찰 → 로그 수집
7. MEMORY_LIMIT=512 → 재실행 → 생존 시간 비교
8. reports/oom.md 작성

[케이스 2: CPU Spike]
9. CPU_MAX_OCCUPY=10 → 실행 → 종료 관찰 → 로그 수집
10. CPU_MAX_OCCUPY=90 → 재실행 → 비교
11. reports/cpu.md 작성

[케이스 3: Deadlock]
12. MULTI_THREAD_ENABLE=true → 실행 → 무응답 관찰
13. ps, top -H, tail -f 로 3가지 증거 수집
14. kill -9 로 강제 종료
15. MULTI_THREAD_ENABLE=false → 재실행 → 정상 동작 비교
16. reports/deadlock.md 작성

17. git add reports/ → git commit → git push
```
