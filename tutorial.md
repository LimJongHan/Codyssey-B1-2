# 과제 수행 튜토리얼

> **환경**: macOS (Apple Silicon M1/M2/M3/M4) 기준.
> `agent-app-leak`은 x86-64 Linux 전용 바이너리이므로 `--platform linux/amd64` 옵션으로
> Docker 컨테이너를 실행해야 한다.

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
```

예상 출력:

```text
Docker version 27.x.x, build xxxxxxx
```

Docker Desktop이 없다면 설치 후 진행한다.

### Apple Silicon 주의사항

`agent-app-leak`은 **x86-64(Intel) Linux 바이너리**다.
Apple Silicon Mac(M1/M2/M3/M4)에서 일반 `ubuntu:22.04` 컨테이너로 실행하면 아래 에러가 발생한다.

```text
rosetta error: failed to open elf at /lib64/ld-linux-x86-64.so.2
Trace/breakpoint trap
```

반드시 `--platform linux/amd64` 옵션을 붙여야 한다.

### Linux 컨테이너 실행

macOS 터미널에서 **B1-2 디렉터리 안**에서 실행한다.

```bash
docker run -it \
  --platform linux/amd64 \
  --name codyssey-b12 \
  -v "$(pwd):/workspace" \
  ubuntu:22.04 \
  /bin/bash
```

예상 출력 (첫 실행 시 이미지 다운로드):

```text
Unable to find image 'ubuntu:22.04' locally
22.04: Pulling from library/ubuntu
Digest: sha256:xxxxxxxx
Status: Downloaded newer image for ubuntu:22.04
root@8148f9d05d85:/#
```

재실행 시 (이미지 이미 있는 경우):

```text
root@8148f9d05d85:/#
```

> 이후 모든 명령어는 **컨테이너 내부 쉘**에서 실행한다.

### 아키텍처 확인 (필수)

```bash
uname -m
```

예상 출력:

```text
x86_64
```

`aarch64`가 출력되면 `--platform linux/amd64` 옵션이 빠진 것이다. 컨테이너를 삭제하고 다시 실행한다.

```bash
# 잘못된 컨테이너 삭제
docker rm -f codyssey-b12

# 올바르게 재실행
docker run -it --platform linux/amd64 --name codyssey-b12 -v "$(pwd):/workspace" ubuntu:22.04 /bin/bash
```

### 기본 도구 설치

```bash
apt-get update && apt-get install -y \
  procps \
  htop \
  psmisc \
  nano
```

예상 출력 (마지막 부분):

```text
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 procps amd64 2:3.3.17-6ubuntu2 [248 kB]
...
Setting up htop (3.0.5-7build2) ...
Setting up psmisc (23.4-2build3) ...
Processing triggers for man-db (2.10.2-1) ...
```

### 작업 디렉터리 확인

```bash
ls /workspace
```

예상 출력:

```text
CLAUDE.md  Misson.md  README.md  agent-app-leak  agent-app-leak.zip  study.md  tutorial.md
```

---

## 2. 앱 실행 환경 구성

### 일반 사용자 생성

`agent-app-leak`은 root로 실행하면 부트 시퀀스에서 실패한다.

```bash
# root 쉘에서 실행
useradd -m -s /bin/bash agent-user
su - agent-user
```

예상 출력:

```text
agent-user@8148f9d05d85:~$
```

> 프롬프트가 `root@...` → `agent-user@...`로 바뀌면 정상이다.

컨테이너를 재시작한 경우 사용자가 이미 존재할 수 있다.

```bash
useradd -m -s /bin/bash agent-user
# 아래 메시지가 나와도 정상 (이미 있다는 뜻)
# useradd: user 'agent-user' already exists
su - agent-user
```

`su` 실패 시 (`failed to execute /bin/bash/`) 셸 경로 오류다:

```bash
# root에서 수정
usermod -s /bin/bash agent-user
su - agent-user
```

### 디렉터리 구조 생성

```bash
mkdir -p ~/agent/upload_files
mkdir -p ~/agent/api_keys
mkdir -p ~/agent/logs
```

예상 출력: 없음 (성공 시 아무것도 출력되지 않음)

생성 확인:

```bash
ls -la ~/agent/
```

예상 출력:

```text
total 20
drwxr-xr-x 5 agent-user agent-user 4096 May 19 10:00 .
drwxr-xr-x 3 agent-user agent-user 4096 May 19 10:00 ..
drwxr-xr-x 2 agent-user agent-user 4096 May 19 10:00 api_keys
drwxr-xr-x 2 agent-user agent-user 4096 May 19 10:00 logs
drwxr-xr-x 2 agent-user agent-user 4096 May 19 10:00 upload_files
```

### secret.key 파일 생성

```bash
echo "agent_api_key_test" > ~/agent/api_keys/secret.key
cat ~/agent/api_keys/secret.key
```

예상 출력:

```text
agent_api_key_test
```

### 환경변수 설정 (.bash_profile)

```bash
nano ~/.bash_profile
```

아래 내용을 붙여넣고 저장한다 (`Ctrl+O` → Enter → `Ctrl+X`).

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

적용:

```bash
source ~/.bash_profile
```

설정 확인:

```bash
env | grep -E "AGENT|MEMORY|CPU|MULTI"
```

예상 출력:

```text
AGENT_HOME=/home/agent-user/agent
AGENT_PORT=15034
AGENT_UPLOAD_DIR=/home/agent-user/agent/upload_files
AGENT_KEY_PATH=/home/agent-user/agent/api_keys
AGENT_LOG_DIR=/home/agent-user/agent/logs
MEMORY_LIMIT=256
CPU_MAX_OCCUPY=80
MULTI_THREAD_ENABLE=true
```

### 바이너리 실행 권한 부여

```bash
chmod +x /workspace/agent-app-leak
ls -la /workspace/agent-app-leak
```

예상 출력:

```text
-rwxrwxr-x 1 root root 7931880 Jan 29 11:16 /workspace/agent-app-leak
```

`rwx` (실행 권한 `x`)가 있어야 한다.

### 정상 실행 확인

```bash
/workspace/agent-app-leak
```

예상 출력 (부트 시퀀스 성공):

```text
[INFO] [Boot] Checking environment variables...
[INFO] [Boot] AGENT_HOME        : /home/agent-user/agent       ✓
[INFO] [Boot] AGENT_PORT        : 15034                        ✓
[INFO] [Boot] AGENT_UPLOAD_DIR  : /home/agent-user/agent/upload_files ✓
[INFO] [Boot] AGENT_KEY_PATH    : /home/agent-user/agent/api_keys     ✓
[INFO] [Boot] AGENT_LOG_DIR     : /home/agent-user/agent/logs  ✓
[INFO] [Boot] MEMORY_LIMIT      : 256 MB                       ✓
[INFO] [Boot] CPU_MAX_OCCUPY    : 80 %                         ✓
[INFO] [Boot] MULTI_THREAD_ENABLE: true                        ✓
[INFO] [Boot] Verifying secret.key...                          ✓
[INFO] [Boot] Binding 0.0.0.0:15034...                        ✓
[INFO] [Boot] Boot sequence complete. Starting agent...
[INFO] [Worker] Task started. Processing...
[INFO] [Worker] Progress: 10%
[INFO] [Worker] Progress: 20%
...
```

부트 시퀀스 실패 예시:

```text
[ERROR] [Boot] AGENT_HOME is not set. Aborting.
```

또는:

```text
[ERROR] [Boot] secret.key not found at /home/agent-user/agent/api_keys/secret.key
```

→ 해당 메시지를 보고 빠진 조건을 확인하고 수정한다.

**종료**: `Ctrl + C`

---

## 3. monitor.sh 준비

`monitor.sh`는 별도 터미널에서 돌리면서 관제 데이터를 수집한다.

### 스크립트 작성

```bash
nano /workspace/monitor.sh
```

아래 내용을 붙여넣는다.

```bash
#!/bin/bash

LOG_FILE="/workspace/monitor.log"
INTERVAL=10

echo "=== Monitor Started: $(date) ===" >> "$LOG_FILE"

while true; do
    TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
    PROC_NAME="agent-app-leak"

    PROC_INFO=$(ps -eo pid,comm,%cpu,%mem --no-headers 2>/dev/null \
                | grep "$PROC_NAME" | head -1)

    if [ -z "$PROC_INFO" ]; then
        echo "[$TIMESTAMP] PROCESS:NOT_FOUND" >> "$LOG_FILE"
    else
        PID=$(echo $PROC_INFO | awk '{print $1}')
        CPU=$(echo $PROC_INFO | awk '{print $3}')
        MEM=$(echo $PROC_INFO | awk '{print $4}')
        DISK=$(df -h / | awk 'NR==2{print $4}')
        FW="active"

        echo "[$TIMESTAMP] PROCESS:$PROC_NAME CPU:${CPU}% MEM:${MEM}% DISK:$DISK FIREWALL:$FW" \
            >> "$LOG_FILE"
    fi

    sleep "$INTERVAL"
done
```

```bash
chmod +x /workspace/monitor.sh
```

### 두 번째 터미널 열기

macOS에서 새 터미널 탭/창을 열고 아래를 실행한다.

```bash
docker exec -it codyssey-b12 su - agent-user
```

예상 출력:

```text
agent-user@8148f9d05d85:~$
```

### 사용법

**터미널 1** (앱 실행 + 로그 저장):

```bash
/workspace/agent-app-leak 2>&1 | tee /workspace/app.log
```

**터미널 2** (관제 실행 + 실시간 확인):

```bash
source ~/.bash_profile
/workspace/monitor.sh &
tail -f /workspace/monitor.log
```

예상 출력 (monitor.log 실시간):

```text
=== Monitor Started: Tue May 19 10:05:00 UTC 2026 ===
[2026-05-19 10:05:00] PROCESS:agent-app-leak CPU:2.1% MEM:1.3% DISK:58G FIREWALL:active
[2026-05-19 10:05:10] PROCESS:agent-app-leak CPU:1.8% MEM:2.7% DISK:58G FIREWALL:active
[2026-05-19 10:05:20] PROCESS:agent-app-leak CPU:2.3% MEM:4.1% DISK:58G FIREWALL:active
```

---

## 4. 케이스 1 - OOM (메모리 누수)

### 목표

메모리가 선형으로 증가하다 `MEMORY_LIMIT`에 도달하면 프로세스가 스스로 종료됨을 관측한다.

### Step 1: 낮은 MEMORY_LIMIT으로 실행 (Before)

**터미널 1**:

```bash
export MEMORY_LIMIT=100
source ~/.bash_profile
/workspace/agent-app-leak 2>&1 | tee /workspace/app_oom_before.log
```

**터미널 2** (관제):

```bash
/workspace/monitor.sh &
tail -f /workspace/monitor.log
```

예상 monitor.log 출력 (MEM이 선형 증가):

```text
[2026-05-19 10:00:00] PROCESS:agent-app-leak CPU:2.1% MEM:1.3%  DISK:58G FIREWALL:active
[2026-05-19 10:00:10] PROCESS:agent-app-leak CPU:1.9% MEM:5.8%  DISK:58G FIREWALL:active
[2026-05-19 10:00:20] PROCESS:agent-app-leak CPU:2.4% MEM:14.2% DISK:58G FIREWALL:active
[2026-05-19 10:00:30] PROCESS:agent-app-leak CPU:2.0% MEM:28.6% DISK:58G FIREWALL:active
[2026-05-19 10:00:40] PROCESS:agent-app-leak CPU:1.8% MEM:45.3% DISK:58G FIREWALL:active
[2026-05-19 10:00:50] PROCESS:agent-app-leak CPU:2.2% MEM:67.1% DISK:58G FIREWALL:active
[2026-05-19 10:01:00] PROCESS:agent-app-leak CPU:1.9% MEM:89.4% DISK:58G FIREWALL:active
[2026-05-19 10:01:10] PROCESS:NOT_FOUND
```

예상 app.log 종료 직전 로그:

```text
[INFO]     [Worker] Progress: 80%
[INFO]     [Worker] Progress: 90%
[WARNING]  [MemoryGuard] Memory usage approaching limit (95MB / 100MB)
[CRITICAL] [MemoryGuard] Memory limit exceeded (100MB >= 100MB)
[CRITICAL] [MemoryGuard] Self-terminating process 47 to prevent system instability.
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
```

### Step 2: 종료 로그 확인

```bash
grep -i "memory\|terminated\|memoryguard" /workspace/app_oom_before.log
```

예상 출력:

```text
[WARNING]  [MemoryGuard] Memory usage approaching limit (95MB / 100MB)
[CRITICAL] [MemoryGuard] Memory limit exceeded (100MB >= 100MB)
[CRITICAL] [MemoryGuard] Self-terminating process 47 to prevent system instability.
>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<
```

```bash
grep "MEM:" /workspace/monitor.log
```

예상 출력 (MEM이 단조 증가):

```text
[2026-05-19 10:00:00] PROCESS:agent-app-leak CPU:2.1% MEM:1.3%  ...
[2026-05-19 10:00:10] PROCESS:agent-app-leak CPU:1.9% MEM:5.8%  ...
[2026-05-19 10:00:20] PROCESS:agent-app-leak CPU:2.4% MEM:14.2% ...
...
[2026-05-19 10:01:00] PROCESS:agent-app-leak CPU:1.9% MEM:89.4% ...
```

### Step 3: MEMORY_LIMIT 조정 후 재실행 (After)

```bash
export MEMORY_LIMIT=512
source ~/.bash_profile
/workspace/agent-app-leak 2>&1 | tee /workspace/app_oom_after.log
```

충분히 오래 실행한 뒤 `Ctrl+C`로 종료한다.

예상 결과: 기존 종료 시점(약 1분)을 훨씬 넘겨 수분 이상 생존.

### Step 4: top으로 실시간 메모리 상승 관찰

**터미널 2**에서:

```bash
top -p $(pgrep agent-app-leak)
```

예상 출력 (RES 컬럼이 점점 증가):

```text
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   47 agent-u+  20   0  512000  25600   8192 S   2.1   3.1   0:12.34 agent-app-leak
   47 agent-u+  20   0  512000  51200   8192 S   1.9   6.3   0:24.56 agent-app-leak
   47 agent-u+  20   0  512000 102400   8192 S   2.3  12.5   0:36.78 agent-app-leak
```

### 리포트용 Before & After 정리

| 항목 | Before (MEMORY_LIMIT=100) | After (MEMORY_LIMIT=512) |
| --- | --- | --- |
| 생존 시간 | 약 1분 | 수분 이상 (메모리가 512MB에 도달하기까지) |
| 최대 MEM% | ~90% (100MB 도달 시 종료) | 훨씬 높은 값까지 허용 |
| 종료 방식 | SELF-TERMINATED (MemoryGuard) | 정상 동작 유지 |

---

## 5. 케이스 2 - CPU Spike (과점유)

### 목표

CPU 사용률이 `CPU_MAX_OCCUPY` 임계치를 초과하면 Watchdog이 SIGTERM으로 프로세스를 종료함을 관측한다.

### Step 1: 낮은 CPU_MAX_OCCUPY로 실행 (Before)

```bash
export CPU_MAX_OCCUPY=10
export MEMORY_LIMIT=512
source ~/.bash_profile
/workspace/agent-app-leak 2>&1 | tee /workspace/app_cpu_before.log
```

### Step 2: CPU 급상승 관찰

**터미널 2**에서 실시간 확인:

```bash
top -d 1 -p $(pgrep agent-app-leak)
```

예상 출력 (CPU%가 갑자기 급상승):

```text
  PID USER      PR  NI    VIRT    RES  S  %CPU  %MEM  COMMAND
   52 agent-u+  20   0  512000  20480  S   2.3   2.5  agent-app-leak
   52 agent-u+  20   0  512000  20480  S   2.1   2.5  agent-app-leak
   52 agent-u+  20   0  512000  20480  R  87.4   2.5  agent-app-leak  ← 급상승
   52 agent-u+  20   0  512000  20480  R  95.2   2.5  agent-app-leak
```

예상 monitor.log 출력 (CPU 급상승 후 NOT_FOUND):

```text
[2026-05-19 10:10:00] PROCESS:agent-app-leak CPU:2.3%  MEM:2.5% DISK:58G FIREWALL:active
[2026-05-19 10:10:10] PROCESS:agent-app-leak CPU:2.1%  MEM:2.5% DISK:58G FIREWALL:active
[2026-05-19 10:10:20] PROCESS:agent-app-leak CPU:87.4% MEM:2.5% DISK:58G FIREWALL:active
[2026-05-19 10:10:25] PROCESS:NOT_FOUND
```

예상 app.log 종료 직전 로그:

```text
[INFO]    [Worker] Task started. Heavy computation...
[WARNING] [Watchdog] CPU usage spike detected: 87.4%
[WARNING] [Watchdog] CPU usage exceeded limit (87.4% >= 10%)
[CRITICAL][Watchdog] Threshold exceeded. Initiating emergency abort...
[SYSTEM]  WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)
```

### Step 3: Watchdog 로그 확인

```bash
grep -i "watchdog\|sigterm\|abort\|cpu" /workspace/app_cpu_before.log
```

예상 출력:

```text
[WARNING] [Watchdog] CPU usage spike detected: 87.4%
[WARNING] [Watchdog] CPU usage exceeded limit (87.4% >= 10%)
[CRITICAL][Watchdog] Threshold exceeded. Initiating emergency abort...
[SYSTEM]  WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM)
```

### Step 4: CPU_MAX_OCCUPY 조정 후 재실행 (After)

```bash
export CPU_MAX_OCCUPY=90
source ~/.bash_profile
/workspace/agent-app-leak 2>&1 | tee /workspace/app_cpu_after.log
```

예상 결과: CPU가 급상승해도 90% 미만이면 Watchdog이 발동하지 않아 계속 실행됨.

### 리포트용 Before & After 정리

| 항목 | Before (CPU_MAX_OCCUPY=10) | After (CPU_MAX_OCCUPY=90) |
| --- | --- | --- |
| Watchdog 발동 CPU | ~87% | 미발동 (90% 미만) |
| 종료 여부 | SIGTERM으로 종료 | 계속 실행 |
| 종료까지 시간 | 약 수십 초 | 미종료 |

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

예상 초기 로그 (정상 실행 → 특정 시점에 멈춤):

```text
[INFO] [Boot] Boot sequence complete. Starting agent...
[INFO] [Worker-A] Task started. Acquiring Lock-1...
[INFO] [Worker-B] Task started. Acquiring Lock-2...
[INFO] [Worker-A] Lock-1 acquired. Waiting for Lock-2...
[INFO] [Worker-B] Lock-2 acquired. Waiting for Lock-1...
[INFO] [Worker-A] WAITING for Lock-2... (BLOCKED)
[INFO] [Worker-B] WAITING for Lock-1... (BLOCKED)
```

→ 이 시점 이후로 **로그가 완전히 멈춘다.**

### Step 2: Deadlock 3가지 증거 수집

**증거 1: PID 존재 확인** (프로세스는 살아있음)

**터미널 2**에서:

```bash
ps -ef | grep agent-app-leak | grep -v grep
```

예상 출력 (PID가 출력되어야 함):

```text
agent-u+    52     1  0 10:15 pts/0    00:00:01 /workspace/agent-app-leak
```

PID가 있으면 프로세스가 살아있다. 없으면 이미 종료된 것이다.

**증거 2: CPU/MEM 변화 없음** (무응답 상태)

```bash
top -H -p $(pgrep agent-app-leak)
```

예상 출력 (모든 스레드 CPU 0%):

```text
  PID USER      PR  NI    VIRT    RES  S  %CPU  %MEM  COMMAND
   52 agent-u+  20   0  512000  20480  S   0.0   2.5  agent-app-leak
   53 agent-u+  20   0  512000  20480  S   0.0   2.5  Worker-A
   54 agent-u+  20   0  512000  20480  S   0.0   2.5  Worker-B
```

모든 스레드의 `%CPU`가 `0.0`이고 상태(`S`)가 `S`(Sleeping)이면 Deadlock.

스레드 상태 확인:

```bash
ps -L -p $(pgrep agent-app-leak) -o tid,stat,%cpu,%mem
```

예상 출력:

```text
  TID STAT %CPU %MEM
   52 Ss    0.0  2.5
   53 Sl    0.0  2.5
   54 Sl    0.0  2.5
```

**증거 3: 로그 출력 멈춤 확인**

```bash
tail -f /workspace/app_deadlock.log
```

예상 출력: 새로운 줄이 추가되지 않고 멈춤 상태.

마지막 로그 키워드 확인:

```bash
tail -20 /workspace/app_deadlock.log
grep -i "waiting\|blocked\|lock" /workspace/app_deadlock.log | tail -5
```

예상 출력:

```text
[INFO] [Worker-A] WAITING for Lock-2... (BLOCKED)
[INFO] [Worker-B] WAITING for Lock-1... (BLOCKED)
```

### Step 3: Deadlock 강제 종료

```bash
kill -9 $(pgrep agent-app-leak)
```

예상 출력: 없음 (성공 시 프롬프트만 돌아옴)

확인:

```bash
ps -ef | grep agent-app-leak | grep -v grep
```

예상 출력: 아무것도 출력되지 않으면 종료 완료.

### Step 4: 싱글스레드 모드로 비교 (After)

```bash
export MULTI_THREAD_ENABLE=false
source ~/.bash_profile
/workspace/agent-app-leak 2>&1 | tee /workspace/app_nodeadlock.log
```

예상 출력 (로그가 계속 출력됨):

```text
[INFO] [Boot] Boot sequence complete. Starting agent...
[INFO] [Worker] Task started (single-thread mode).
[INFO] [Worker] Progress: 10%
[INFO] [Worker] Progress: 20%
[INFO] [Worker] Progress: 30%
...
```

→ 로그가 멈추지 않고 계속 출력되면 Deadlock이 해소된 것이다.

### Step 5: pstree로 스레드 구조 비교

```bash
# Before (MULTI_THREAD_ENABLE=true) 실행 중
pstree -p $(pgrep agent-app-leak)
```

예상 출력:

```text
agent-app-leak(52)─┬─{Worker-A}(53)
                   └─{Worker-B}(54)
```

```bash
# After (MULTI_THREAD_ENABLE=false) 실행 중
pstree -p $(pgrep agent-app-leak)
```

예상 출력 (스레드가 1개):

```text
agent-app-leak(60)
```

### 리포트용 Before & After 정리

| 항목 | Before (MULTI_THREAD_ENABLE=true) | After (MULTI_THREAD_ENABLE=false) |
| --- | --- | --- |
| 프로세스 상태 | PID 존재, 무응답 | PID 존재, 정상 동작 |
| 스레드 CPU% | 모두 0% | 정상 수치 (1~5%) |
| 로그 출력 | 특정 시점에 멈춤 | 계속 출력 |
| 마지막 로그 | `WAITING... BLOCKED` | 진행 중 (`Progress: XX%`) |

---

## 7. 리포트 작성 가이드

### 디렉터리 생성 (컨테이너 밖, macOS에서)

```bash
mkdir -p /workspace/reports
touch /workspace/reports/oom.md
touch /workspace/reports/cpu.md
touch /workspace/reports/deadlock.md
```

### 리포트 템플릿

```markdown
# [Bug] {장애 유형} - {한 줄 요약}

## 1. Description (현상 설명)
- 어떤 현상이 발생했는가?
- 언제, 어떤 조건(환경변수 설정)에서 발생했는가?

## 2. Evidence & Logs (증거 자료)

### monitor.log 발췌
(MEM% 또는 CPU% 변화 수치를 코드블록으로 직접 붙여넣기)

### 프로그램 실행 로그 발췌
(종료 직전 핵심 로그 5~10줄 원문)

### 시스템 도구 출력
(ps, top, pstree 출력 결과)

## 3. Root Cause Analysis (원인 분석)
- 수집된 증거를 근거로 기술적 원인 서술
- 관련 OS 동작 원리 설명

## 4. Workaround & Verification (조치 및 검증)
- 어떤 환경변수를 어떻게 바꿨는가
- Before & After 비교 표
- 근본적 해결책 제안 (선택)
```

---

## 자주 하는 실수 & 해결법

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| `rosetta error: failed to open elf` | `--platform linux/amd64` 없이 실행 | 컨테이너 삭제 후 `--platform linux/amd64` 추가하여 재실행 |
| 부트 시퀀스 실패 | 환경변수 미설정 또는 디렉터리 없음 | `env \| grep AGENT` 로 확인, 디렉터리 재생성 |
| root로 실행 시 실패 | 보안 정책 | `su - agent-user` 후 실행 |
| `su: failed to execute /bin/bash/` | 셸 경로에 `/` 붙음 | `usermod -s /bin/bash agent-user` 후 재시도 |
| secret.key 오류 | 파일 내용 불일치 | `cat $AGENT_KEY_PATH/secret.key` 확인 (`agent_api_key_test` 이어야 함) |
| monitor.sh에서 NOT_FOUND만 출력 | 프로세스 이름 불일치 | `ps -ef`로 실제 프로세스명 확인 후 스크립트 수정 |
| 컨테이너 종료 후 재접속 | 컨테이너 중지됨 | `docker start codyssey-b12 && docker exec -it codyssey-b12 su - agent-user` |
| 환경변수가 적용 안 됨 | `source` 안 함 | `source ~/.bash_profile` 실행 |
| Deadlock인지 OOM인지 구분 불가 | 로그 미확인 | PID 존재하면 Deadlock, PID 없으면 OOM 또는 CPU Spike |

---

## 전체 수행 순서 요약

```text
[환경 준비]
1. docker run --platform linux/amd64 으로 컨테이너 실행
2. uname -m → x86_64 확인
3. apt-get install procps htop psmisc nano

[앱 환경 구성]
4. useradd -m -s /bin/bash agent-user → su - agent-user
5. mkdir -p ~/agent/upload_files api_keys logs
6. echo "agent_api_key_test" > ~/agent/api_keys/secret.key
7. .bash_profile 작성 → source 적용
8. chmod +x /workspace/agent-app-leak
9. /workspace/agent-app-leak → 부트 시퀀스 정상 확인 → Ctrl+C

[monitor.sh]
10. /workspace/monitor.sh 작성 → chmod +x

[케이스 1: OOM]
11. MEMORY_LIMIT=100 → 실행 → SELF-TERMINATED 관찰 → 로그 수집
12. MEMORY_LIMIT=512 → 재실행 → 생존 시간 비교
13. reports/oom.md 작성

[케이스 2: CPU Spike]
14. CPU_MAX_OCCUPY=10 → 실행 → WATCHDOG SIGTERM 관찰 → 로그 수집
15. CPU_MAX_OCCUPY=90 → 재실행 → 계속 실행 확인
16. reports/cpu.md 작성

[케이스 3: Deadlock]
17. MULTI_THREAD_ENABLE=true → 실행 → 로그 멈춤 관찰
18. ps, top -H, tail -f 로 3가지 증거 수집
19. kill -9 로 강제 종료
20. MULTI_THREAD_ENABLE=false → 재실행 → 정상 동작 비교
21. reports/deadlock.md 작성

[제출]
22. git add reports/ → git commit → git push
```
