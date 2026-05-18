# 리눅스 트러블슈팅 학습 가이드

> 비전공자를 위한 기초부터 실전까지. 이 문서 하나로 미션에 필요한 모든 개념을 커버한다.

---

## 목차

1. [운영체제(OS) 기초](#1-운영체제os-기초)
2. [프로세스와 스레드](#2-프로세스와-스레드)
3. [메모리 구조와 메모리 누수](#3-메모리-구조와-메모리-누수)
4. [CPU와 스케줄링](#4-cpu와-스케줄링)
5. [교착상태(Deadlock)](#5-교착상태deadlock)
6. [리눅스 기본 지식](#6-리눅스-기본-지식)
7. [환경변수](#7-환경변수)
8. [모니터링 명령어 실전](#8-모니터링-명령어-실전)
9. [시그널(Signal)](#9-시그널signal)
10. [트러블슈팅 사고법](#10-트러블슈팅-사고법)

---

## 1. 운영체제(OS) 기초

### 운영체제란 무엇인가?

컴퓨터는 크게 **하드웨어**(CPU, RAM, 디스크 등)와 **소프트웨어**(프로그램)로 나뉜다.
운영체제(OS, Operating System)는 그 중간에 위치하여 하드웨어를 관리하고, 프로그램들이 하드웨어를 안전하게 사용할 수 있도록 중재하는 소프트웨어다.

```text
[사용자 프로그램]  ← 우리가 만드는 것
      ↕
[운영체제 (OS)]    ← Linux, Windows, macOS
      ↕
[하드웨어]         ← CPU, RAM, 디스크, 네트워크 카드
```

### 왜 OS가 필요한가?

예를 들어 프로그램 A와 프로그램 B가 동시에 실행된다고 하자.
OS 없이는 두 프로그램이 같은 메모리 공간을 덮어쓰거나, CPU를 영원히 독점할 수 있다.
OS는 이를 방지하여 **자원을 공평하고 안전하게 분배**한다.

### 리눅스(Linux)란?

리눅스는 오픈소스 운영체제다. 서버 환경에서 압도적으로 많이 쓰인다.
우리가 사용하는 웹서비스(유튜브, 네이버, 카카오 등)의 서버는 대부분 리눅스 위에서 돌아간다.

**커널(Kernel)**: OS의 핵심 부분. 하드웨어와 소프트웨어 사이에서 실제로 자원을 관리한다.

---

## 2. 프로세스와 스레드

### 프로세스(Process)란?

**프로그램**: 디스크에 저장된 실행 파일 (`.exe`, 바이너리 파일 등) → 정지된 상태
**프로세스**: 그 프로그램이 메모리에 올라와 실행 중인 상태 → 살아있는 상태

```text
agent-leak-app (파일)  →  실행  →  agent-leak-app 프로세스 (메모리에서 동작)
```

프로세스는 실행되는 순간 OS로부터 고유한 **PID(Process ID)** 번호를 부여받는다.

```text
PID 1    → systemd (리눅스 시작 프로세스)
PID 1234 → agent-leak-app
PID 5678 → python3
```

### 프로세스가 갖는 자원

프로세스가 실행되면 OS는 아래 자원들을 할당해준다.

| 자원 | 설명 |
| --- | --- |
| **메모리 공간** | 코드, 데이터, 스택, 힙 영역 |
| **CPU 시간** | CPU를 사용할 수 있는 시간 할당량 |
| **파일 디스크립터** | 열린 파일, 소켓 등 |
| **PID** | 프로세스 고유 식별 번호 |

### 프로세스 생명주기

```text
생성(Created) → 준비(Ready) → 실행(Running) → 대기(Waiting) → 종료(Terminated)
                    ↑___________________↓
```

- **생성**: `./agent-leak-app` 명령 입력
- **준비**: CPU를 기다리는 중
- **실행**: 실제로 CPU를 사용하며 코드가 돌아가는 중
- **대기**: I/O(파일 읽기, 네트워크 등) 완료를 기다리는 중
- **종료**: 정상 종료 또는 강제 종료

### 스레드(Thread)란?

프로세스 하나 안에서 동시에 여러 작업을 처리하기 위한 실행 단위.
프로세스가 **공장** 이라면, 스레드는 그 공장 안에서 일하는 **작업자** 다.

```text
[프로세스: agent-leak-app]
  ├── Thread-A: 파일 업로드 처리
  ├── Thread-B: API 요청 처리
  └── Thread-C: 로그 기록
```

**핵심**: 같은 프로세스 내의 스레드들은 **메모리를 공유**한다. 이것이 Deadlock의 원인이 된다.

### 멀티스레드의 장단점

| 장점 | 단점 |
| --- | --- |
| 동시에 여러 작업 처리 가능 | 공유 자원 충돌 위험 |
| 빠른 응답성 | Deadlock 발생 가능 |
| 자원 효율적 사용 | 디버깅이 복잡 |

---

## 3. 메모리 구조와 메모리 누수

### RAM이란?

RAM(Random Access Memory)은 현재 실행 중인 프로그램과 데이터를 임시로 저장하는 공간이다.
컴퓨터 전원을 끄면 사라지는 **휘발성 메모리**.

디스크(SSD/HDD)는 영구 저장소이지만 속도가 느리다.
CPU는 디스크에서 직접 읽지 않고, 반드시 RAM을 통해 데이터를 가져온다.

```text
속도: CPU 캐시 >>> RAM >>> SSD >>> HDD
용량: CPU 캐시 <<< RAM <<< SSD <<< HDD
```

### 프로세스의 메모리 구조

프로세스 하나가 실행되면 OS는 다음과 같이 메모리를 구분하여 할당한다.

```text
높은 주소
┌─────────────┐
│   Stack     │  ← 함수 호출, 지역 변수 저장 (자동 관리)
│      ↓      │
│             │
│      ↑      │
│    Heap     │  ← 동적으로 할당하는 메모리 (프로그래머가 관리)
├─────────────┤
│    Data     │  ← 전역 변수, 정적 변수
├─────────────┤
│    Code     │  ← 실행할 프로그램 코드
낮은 주소
```

### 힙(Heap)이 핵심인 이유

- **스택**: 함수가 끝나면 OS가 자동으로 메모리를 회수한다.
- **힙**: 프로그래머가 직접 할당하고, **직접 해제해야** 한다. 해제하지 않으면 메모리가 계속 쌓인다.

```python
# Python 예시 (내부적으로 힙 사용)
data = []
while True:
    data.append("A" * 1000000)  # 계속 추가만 하고 삭제 안 함 → 메모리 누수
```

### 메모리 누수(Memory Leak)란?

프로그램이 더 이상 사용하지 않는 메모리를 해제하지 않아, **사용 가능한 메모리가 시간이 지남에 따라 계속 줄어드는 현상**.

```text
시간 →
메모리 사용량: 10MB → 50MB → 150MB → 300MB → 💥 종료
```

**실생활 비유**: 쓰레기통을 비우지 않는 것. 쓰레기가 계속 쌓이다가 결국 집 전체가 쓰레기로 꽉 찬다.

### OOM(Out Of Memory)이란?

메모리 누수가 계속되어 프로세스가 사용 가능한 메모리를 모두 소진한 상태.
이때 OS 또는 애플리케이션 내부 정책이 해당 프로세스를 강제 종료한다.

```text
[Linux OOM Killer]: OS가 직접 프로세스를 죽임
[MemoryGuard]:      agent-leak-app 내부 정책이 스스로 종료 (SELF-TERMINATED)
```

### 이번 미션에서의 메모리 누수 흐름

```text
agent-leak-app 실행
     ↓
메모리 사용량 점진적 증가 (monitor.sh로 관찰)
     ↓
MEMORY_LIMIT 도달 (예: 256MB)
     ↓
[CRITICAL] MemoryGuard 발동
     ↓
SELF-TERMINATED
```

### 메모리 관련 핵심 지표

- **RSS (Resident Set Size)**: 실제로 RAM에 올라와 있는 메모리 크기. `ps`, `top`에서 확인 가능.
- **VSZ (Virtual Size)**: 프로세스가 예약한 가상 메모리 크기. RSS보다 항상 크다.
- **%MEM**: 전체 RAM 대비 해당 프로세스의 사용 비율.

---

## 4. CPU와 스케줄링

### CPU란?

CPU(Central Processing Unit)는 컴퓨터의 두뇌.
코드를 실제로 **연산**하는 장치. 코어가 많을수록 동시에 더 많은 작업을 처리 가능.

### CPU 사용률이란?

CPU가 얼마나 바쁜지를 나타내는 지표. 100%이면 CPU가 쉬지 못하고 모든 자원을 쏟아붓는 상태.

```text
CPU 사용률 0%:   CPU가 놀고 있음
CPU 사용률 50%:  절반만 사용 중
CPU 사용률 100%: CPU 풀가동, 다른 프로세스가 처리되지 못함
```

### CPU 스케줄링이란?

CPU는 한 번에 하나의 작업만 처리한다(코어 하나 기준).
여러 프로세스가 동시에 실행되는 것처럼 보이는 이유는 **CPU가 매우 빠르게 번갈아가며 각 프로세스를 처리**하기 때문이다.

이 순서와 방법을 결정하는 것이 **CPU 스케줄링 알고리즘**이다.

### 스케줄링 알고리즘 종류

#### FCFS (First Come, First Served) - 선착순

먼저 온 프로세스를 먼저 끝까지 처리한다.

```text
도착 순서: A → B → C
실행 순서: [AAAA][BBBB][CCCC]
```

- **장점**: 단순하고 공평
- **단점**: A가 오래 걸리면 B, C가 오래 기다려야 함 (convoy effect)
- **적합한 서비스**: 배치 처리, 순서가 중요한 작업

#### Round-Robin (라운드 로빈) - 시간 할당

각 프로세스에 동일한 시간(Time Quantum)을 할당하고 돌아가며 처리한다.

```text
Time Quantum = 50ms
실행 순서: [A 50ms][B 50ms][C 50ms][A 50ms][B 50ms]...
```

- **장점**: 모든 프로세스가 공평하게 CPU를 사용
- **단점**: 컨텍스트 스위칭 오버헤드 발생
- **적합한 서비스**: 웹 서버, 실시간 응답이 중요한 시스템

#### Priority Scheduling - 우선순위

우선순위가 높은 프로세스를 먼저 처리한다.

```text
우선순위: A(3) < B(5) < C(10)
실행 순서: [CCCC][BBBB][AAAA]
```

- **장점**: 중요한 작업을 빠르게 처리
- **단점**: 낮은 우선순위 프로세스는 영원히 실행 못 할 수 있음 (starvation)
- **적합한 서비스**: 실시간 시스템, 긴급 처리가 필요한 환경

### CPU 과점유(CPU Spike)란?

특정 프로세스가 CPU를 과도하게 점유하여 다른 프로세스가 실행되지 못하는 상태.

```text
정상:  agent-leak-app CPU: 5~20%
과점유: agent-leak-app CPU: 95~100% → 시스템 전체 응답 불가
```

### 이번 미션에서의 CPU 과점유 흐름

```text
agent-leak-app 실행
     ↓
특정 시점에 CPU 사용률 급상승
     ↓
CPU_MAX_OCCUPY 임계치 초과
     ↓
[Watchdog] 과점유 방지 정책 발동
     ↓
SIGTERM 전송 → 프로세스 종료
```

### 컨텍스트 스위칭(Context Switching)

CPU가 한 프로세스에서 다른 프로세스로 전환할 때, 현재 상태(레지스터, 프로그램 카운터 등)를 저장하고 새 프로세스의 상태를 불러오는 작업.
**오버헤드**가 발생하므로 스위칭이 너무 잦으면 성능이 떨어진다.

---

## 5. 교착상태(Deadlock)

### Deadlock이란?

두 개 이상의 스레드/프로세스가 **서로 상대방이 가진 자원을 기다리며 무한히 멈춰있는 상태**.

### 식사하는 철학자 문제

가장 유명한 Deadlock 예시.

```text
      [철학자 1]
    젓가락1   젓가락2
[철학자 4]         [철학자 2]
    젓가락4   젓가락3
      [철학자 3]
```

- 식사하려면 왼쪽과 오른쪽 젓가락 **둘 다** 필요하다.
- 모든 철학자가 동시에 왼쪽 젓가락을 집었다.
- 이제 모두 오른쪽 젓가락을 기다린다.
- 아무도 오른쪽 젓가락을 내려놓지 않는다.
- **영원히 아무도 식사하지 못한다** → Deadlock

### Deadlock의 4대 조건

아래 4가지가 **동시에** 성립할 때 Deadlock이 발생한다. 하나라도 깨면 Deadlock은 발생하지 않는다.

| 조건 | 설명 | 예시 |
| --- | --- | --- |
| **상호 배제** (Mutual Exclusion) | 자원은 한 번에 하나의 스레드만 사용 가능 | 젓가락은 한 명만 쥘 수 있음 |
| **점유 대기** (Hold and Wait) | 자원을 가진 채로 다른 자원을 기다림 | 왼쪽 젓가락을 쥔 채 오른쪽 기다림 |
| **비선점** (No Preemption) | 다른 스레드의 자원을 강제로 빼앗을 수 없음 | 남의 젓가락을 뺏을 수 없음 |
| **순환 대기** (Circular Wait) | A→B→C→A 형태로 서로 기다리는 순환 형성 | 1→2→3→4→1 순으로 기다림 |

### 코드로 이해하는 Deadlock

```text
Thread-A가 하는 일:
  1. Lock-1 획득
  2. Lock-2 획득 시도 (Lock-2는 Thread-B가 점유 중) → 무한 대기

Thread-B가 하는 일:
  1. Lock-2 획득
  2. Lock-1 획득 시도 (Lock-1은 Thread-A가 점유 중) → 무한 대기
```

**결과**: Thread-A와 Thread-B 모두 영원히 대기 → 프로세스 무응답

### Deadlock 식별 방법

프로세스가 Deadlock 상태이면:

| 지표 | 상태 |
| --- | --- |
| PID | **존재** (프로세스는 살아있음) |
| CPU 사용률 | **0% 또는 매우 낮음** (아무것도 안 함) |
| 메모리 사용량 | **변화 없음** |
| 로그 출력 | **완전히 멈춤** |

```bash
# PID는 있지만 CPU 0%
ps -ef | grep agent-leak-app   # PID 존재 확인
top -H -p <PID>                # 스레드별 CPU 확인 → 모두 0%
```

### 이번 미션에서의 Deadlock 흐름

```text
MULTI_THREAD_ENABLE=true → 멀티스레드 모드 실행
     ↓
Thread-A, Thread-B 동시 실행
     ↓
서로의 Lock을 기다리는 순환 대기 발생
     ↓
로그 출력 멈춤 ("WAITING… BLOCKED")
     ↓
CPU: 0%, MEM: 변화 없음, PID는 존재
```

**해결**: `MULTI_THREAD_ENABLE=false` → 싱글스레드 모드로 전환, 자원 경쟁 없음

---

## 6. 리눅스 기본 지식

### 파일 시스템 구조

리눅스는 모든 것이 파일이다. 디렉터리 구조는 `/`(루트)에서 시작한다.

```text
/
├── home/         ← 일반 사용자 홈 디렉터리 (/home/jonghan)
├── root/         ← root 사용자 홈
├── etc/          ← 설정 파일들
├── var/          ← 로그, 임시 파일
├── tmp/          ← 임시 파일 (재부팅 시 삭제)
├── usr/          ← 프로그램, 라이브러리
└── bin/          ← 기본 명령어들 (ls, cp, mv 등)
```

### 자주 쓰는 기본 명령어

```bash
# 현재 위치 확인
pwd

# 디렉터리 이동
cd /home/jonghan
cd ..        # 상위 디렉터리
cd ~         # 홈 디렉터리

# 파일/디렉터리 목록
ls           # 기본 목록
ls -l        # 상세 정보 (권한, 크기, 날짜)
ls -la       # 숨김 파일 포함

# 파일 내용 보기
cat filename.log
tail -f filename.log    # 실시간으로 끝 부분 보기 (로그 모니터링에 유용)
head -n 50 filename.log # 처음 50줄만 보기

# 디렉터리 생성
mkdir mydir
mkdir -p a/b/c    # 중간 디렉터리도 함께 생성

# 파일 복사/이동/삭제
cp source.txt dest.txt
mv old.txt new.txt
rm filename.txt
rm -rf directory/   # 디렉터리 통째로 삭제 (주의!)

# 파일 검색
grep "CRITICAL" app.log           # 로그에서 CRITICAL 포함 줄 찾기
grep -n "MEMORY" app.log          # 줄 번호 포함
grep -A 5 "SELF-TERMINATED" app.log  # 해당 줄 + 아래 5줄

# 권한 변경
chmod +x script.sh    # 실행 권한 부여
chmod 755 script.sh   # 숫자로 권한 설정

# 현재 사용자 확인
whoami
```

### 파일 권한 이해

```bash
-rwxr-xr-x  1  jonghan  staff  12345  May 18  agent-leak-app
 ↑↑↑↑↑↑↑↑↑     ↑↑↑↑↑↑↑
 │└─┬─┘└─┬─┘   소유자  그룹
 │  │    └── 그 외 사용자 권한 (r-x: 읽기, 실행만 가능)
 │  └──────── 그룹 권한 (r-x: 읽기, 실행만 가능)
 └─────────── 소유자 권한 (rwx: 읽기, 쓰기, 실행 모두 가능)
```

| 문자 | 의미 |
| --- | --- |
| `r` | read (읽기) |
| `w` | write (쓰기) |
| `x` | execute (실행) |
| `-` | 해당 권한 없음 |

### 사용자와 root

- **일반 사용자**: 제한된 권한. 자신의 파일만 수정 가능.
- **root**: 슈퍼유저. 모든 권한 보유. 잘못 쓰면 시스템 전체에 영향.

> 이번 미션에서 `agent-leak-app`은 **root가 아닌 일반 사용자로 실행**해야 한다. 이는 보안 정책이다.

```bash
whoami       # 현재 사용자 확인 → root가 아니어야 함
sudo command # root 권한으로 명령 실행
```

### 프로세스 실행 방법

```bash
# 포그라운드 실행 (터미널 점유)
./agent-leak-app

# 백그라운드 실행 (터미널 반환)
./agent-leak-app &

# 실행 중인 백그라운드 작업 확인
jobs

# 백그라운드 → 포그라운드
fg %1

# 출력을 파일로 저장
./agent-leak-app > app.log 2>&1
./agent-leak-app >> app.log 2>&1    # 이어쓰기
```

---

## 7. 환경변수

### 환경변수란?

프로그램이 실행될 때 참조할 수 있는 **이름=값** 형태의 설정값.
코드를 수정하지 않고 프로그램의 동작을 바꿀 수 있다.

```bash
# 환경변수 설정 (현재 터미널 세션에서만 유효)
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=80
export MULTI_THREAD_ENABLE=true

# 환경변수 확인
echo $MEMORY_LIMIT        # 256 출력
printenv MEMORY_LIMIT     # 256 출력
env                       # 모든 환경변수 출력

# 환경변수 삭제
unset MEMORY_LIMIT
```

### .bash_profile이란?

터미널을 열 때마다 자동으로 실행되는 설정 파일.
여기에 `export`를 넣어두면 **영구적으로** 환경변수가 설정된다.

```bash
# ~/.bash_profile 편집
vi ~/.bash_profile
# 또는
nano ~/.bash_profile

# 파일 내용 예시
export AGENT_HOME=/home/jonghan/agent
export AGENT_PORT=15034
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=80
export MULTI_THREAD_ENABLE=true

# 수정 후 적용
source ~/.bash_profile
```

### 이번 미션의 환경변수 정리

| 환경변수 | 역할 | 범위 |
| --- | --- | --- |
| `AGENT_HOME` | 앱의 기준 디렉터리 | 경로 |
| `AGENT_PORT` | 서버 포트 | 15034 (고정) |
| `AGENT_UPLOAD_DIR` | 업로드 파일 경로 | `$AGENT_HOME/upload_files` |
| `AGENT_KEY_PATH` | API 키 경로 | `$AGENT_HOME/api_keys` |
| `AGENT_LOG_DIR` | 로그 저장 경로 | 쓰기 권한 필요 |
| `MEMORY_LIMIT` | 메모리 상한선 | 50~512 (MB) |
| `CPU_MAX_OCCUPY` | CPU 점유율 상한 | 10~100 (%) |
| `MULTI_THREAD_ENABLE` | 멀티스레드 활성화 | true/false |

### 환경변수로 트러블슈팅하기

```bash
# Before: MEMORY_LIMIT=256 → 10분 만에 종료
export MEMORY_LIMIT=256
./agent-leak-app
# → SELF-TERMINATED 발생

# After: MEMORY_LIMIT=512 → 더 오래 생존
export MEMORY_LIMIT=512
./agent-leak-app
# → 30분 이상 생존
```

---

## 8. 모니터링 명령어 실전

### ps - 프로세스 스냅샷

현재 실행 중인 프로세스 목록을 보여준다. (실시간 아님, 한 번 찍는 스냅샷)

```bash
# 모든 프로세스 표시
ps -ef

# 특정 프로세스 찾기
ps -ef | grep agent-leak-app

# 결과 예시
# UID   PID  PPID  C  STIME  TTY  TIME     CMD
# user  1234  1000  5  10:00  ?    00:01:23 ./agent-leak-app
```

| 컬럼 | 의미 |
| --- | --- |
| `UID` | 실행 사용자 |
| `PID` | 프로세스 ID |
| `PPID` | 부모 프로세스 ID |
| `C` | CPU 사용률 |
| `STIME` | 시작 시간 |
| `CMD` | 실행 명령어 |

```bash
# 스레드까지 표시 (Deadlock 확인 시 사용)
ps -L -p <PID>        # 특정 PID의 스레드 목록
ps -eLf | grep agent  # 모든 스레드 표시
```

### top - 실시간 모니터링

CPU, 메모리 사용률을 **실시간**으로 보여준다. `q`를 눌러 종료.

```bash
top
top -p 1234         # 특정 PID만 모니터링
top -H -p 1234      # 특정 PID의 스레드별 모니터링 (Deadlock 확인)
```

**top 화면 읽는 법**

```text
top - 10:30:00 up 2 days  load average: 0.52, 0.45, 0.40
Tasks:  95 total,   1 running,  94 sleeping
%Cpu(s):  5.2 us,  1.1 sy,  0.0 ni, 93.4 id

  PID USER   PR  NI  VIRT   RES  SHR  S  %CPU  %MEM  TIME+    COMMAND
 1234 user   20   0  512m  256m   8m  S  95.0   5.0   1:23.45  agent-leak-app
```

| 항목 | 의미 |
| --- | --- |
| `load average` | 최근 1분/5분/15분 평균 부하. 코어 수보다 높으면 과부하 |
| `%Cpu id` | CPU 유휴(idle) 비율. 낮을수록 바쁨 |
| `PID` | 프로세스 ID |
| `%CPU` | CPU 사용률 |
| `%MEM` | 메모리 사용률 |
| `RES` | 실제 RAM 사용량 (RSS) |
| `S` | 상태 (R=실행, S=슬립, D=디스크대기, Z=좀비) |

**top 내에서 유용한 키**

| 키 | 기능 |
| --- | --- |
| `M` | 메모리 사용량 기준 정렬 |
| `P` | CPU 사용량 기준 정렬 |
| `k` | 특정 PID에 시그널 전송 (kill) |
| `1` | CPU 코어별 사용률 표시 |
| `H` | 스레드 표시 모드 토글 |

### htop - top의 개선판

top보다 더 보기 좋고 직관적인 모니터링 도구.

```bash
htop
htop -p 1234     # 특정 PID만
```

- 방향키로 프로세스 선택
- `F9`로 시그널 전송
- `F6`으로 정렬 기준 변경

### pstree - 프로세스 트리

부모-자식 관계를 트리 형태로 시각화.

```bash
pstree             # 전체 트리
pstree -p          # PID 포함
pstree -p 1234     # 특정 PID의 트리
```

```text
# 출력 예시
systemd(1)─┬─agent-leak-app(1234)─┬─{Thread-A}(1235)
           │                      ├─{Thread-B}(1236)
           │                      └─{Thread-C}(1237)
```

### monitor.sh - 관제 스크립트

이번 미션에서 제공하는 스크립트. 일정 간격으로 CPU/MEM/DISK 등을 기록.

```bash
./monitor.sh         # 실행
```

**로그 형식 이해**

```text
[2025-12-30 14:00:00] PROCESS:agent-leak-app CPU:1.2% MEM:5.1% DISK:954G FIREWALL:active
```

| 항목 | 의미 |
| --- | --- |
| 타임스탬프 | 측정 시각 |
| `CPU` | CPU 사용률 |
| `MEM` | 메모리 사용률 |
| `DISK` | 사용 가능 디스크 용량 |
| `FIREWALL` | 방화벽 상태 |

### grep으로 로그 분석

```bash
# 특정 키워드 찾기
grep "CRITICAL" app.log
grep "SELF-TERMINATED" app.log
grep "WATCHDOG" app.log
grep "BLOCKED" app.log

# 여러 키워드 동시 검색
grep -E "CRITICAL|TERMINATED|WATCHDOG" app.log

# 키워드 전후 줄 함께 보기
grep -B 5 -A 5 "SELF-TERMINATED" app.log  # 전 5줄, 후 5줄

# 타임스탬프 순으로 패턴 파악
grep "MEM:" monitor.log | tail -20  # 최근 20개 메모리 수치
```

---

## 9. 시그널(Signal)

### 시그널이란?

OS가 프로세스에게 보내는 **비동기 메시지**. 프로세스의 동작을 제어할 때 사용.

### 주요 시그널

| 시그널 | 번호 | 의미 | 처리 가능 여부 |
| --- | --- | --- | --- |
| `SIGTERM` | 15 | 정상 종료 요청 | 가능 (무시할 수 있음) |
| `SIGKILL` | 9 | 강제 종료 | 불가 (무조건 종료) |
| `SIGINT` | 2 | 인터럽트 (Ctrl+C) | 가능 |
| `SIGHUP` | 1 | 터미널 끊김 | 가능 |

### kill 명령어

```bash
# 프로세스에 시그널 보내기
kill <PID>           # 기본값: SIGTERM (정상 종료 요청)
kill -9 <PID>        # SIGKILL (강제 종료)
kill -15 <PID>       # SIGTERM (명시적)

# 이름으로 종료
pkill agent-leak-app
killall agent-leak-app
```

### SIGTERM vs SIGKILL

```text
SIGTERM (정상 종료):
  프로세스에게 "끝내도 됩니다" 메시지 전송
  프로세스가 받아서 정리 작업(로그 저장, 파일 닫기 등) 후 종료
  프로세스가 무시하면 종료 안 될 수 있음

SIGKILL (강제 종료):
  OS가 직접 프로세스를 즉시 종료
  프로세스가 어떤 상태여도 무조건 종료
  정리 작업 없음 → 데이터 손실 가능
```

**이번 미션 관련**:

- Watchdog이 `SIGTERM`을 보냄 → 앱이 로그 남기고 종료
- MemoryGuard가 `SIGKILL`로 강제 종료 가능 (SELF-TERMINATED)

---

## 10. 트러블슈팅 사고법

### 관측 → 증거 → 원인 → 조치 → 검증

모든 장애 분석은 이 5단계 흐름을 따른다.

```text
1. 관측: "뭔가 이상하다"
   → 프로세스가 갑자기 종료됨

2. 증거 수집: "무슨 일이 있었나?"
   → monitor.sh 로그, 프로그램 로그, ps/top 출력

3. 원인 분석: "왜 이런 일이 생겼나?"
   → 메모리 선형 증가 패턴 + MEMORY_LIMIT 도달 로그

4. 조치: "어떻게 해결하나?"
   → MEMORY_LIMIT 값 조정

5. 검증: "조치가 효과 있었나?"
   → Before(256MB, 10분 생존) vs After(512MB, 30분 생존)
```

### 장애 유형별 1차 체크리스트

#### OOM 의심 시

```bash
# 1. 프로세스가 종료되었는지 확인
ps -ef | grep agent-leak-app   # 없으면 종료됨

# 2. 로그에서 메모리 관련 메시지 확인
grep -i "memory\|TERMINATED\|MemoryGuard" app.log

# 3. monitor.sh 로그에서 MEM 수치 추이 확인
grep "MEM:" monitor.log

# 4. 시스템 OOM 로그 확인
dmesg | grep -i "oom\|out of memory"
```

#### CPU Spike 의심 시

```bash
# 1. 현재 CPU 사용률 확인
top -p $(pgrep agent-leak-app)

# 2. 로그에서 Watchdog 메시지 확인
grep -i "watchdog\|SIGTERM\|cpu" app.log

# 3. monitor.sh에서 CPU 급상승 구간 찾기
grep "CPU:" monitor.log | grep -v "CPU:[0-9]\."
```

#### Deadlock 의심 시

```bash
# 1. PID 존재 확인 (살아있는지)
ps -ef | grep agent-leak-app   # 있으면 살아있음

# 2. CPU/MEM 변화 없는지 확인 (무응답 상태)
top -H -p $(pgrep agent-leak-app)   # 스레드별 CPU → 모두 0%

# 3. 로그 멈춤 확인
tail -f app.log   # 새로운 로그가 안 나오면 멈춘 것

# 4. 마지막 로그 확인
tail -50 app.log
grep -i "waiting\|blocked\|lock" app.log | tail -10
```

### 로그를 읽는 습관

로그는 시간순으로 쌓인다. 장애 분석 시:

1. **종료 직전 로그**를 먼저 확인 (`tail -100 app.log`)
2. **키워드로 필터링** (`grep "CRITICAL\|ERROR\|WARN" app.log`)
3. **타임스탬프로 시간대 맞추기** (monitor.log와 app.log의 시간 대조)

### GitHub Issue 작성 팁

좋은 기술 리포트는 **재현 가능**해야 한다. 읽는 사람이 동일한 절차를 따라했을 때 같은 현상을 볼 수 있어야 한다.

```markdown
나쁜 예: "메모리가 많이 쓰여서 종료되었다."

좋은 예: "MEMORY_LIMIT=256 설정 하에 agent-leak-app을 실행한 지 약 10분 후,
monitor.log에서 MEM이 96.8%까지 상승한 것이 확인되었으며,
동 시각 app.log에서 [CRITICAL] MemoryGuard Memory limit exceeded (256MB >= 256MB) 로그와 함께
SELF-TERMINATED 메시지가 출력되며 프로세스가 종료되었다."
```

**핵심 원칙**: 수치, 로그 문자열, 시각 등 **객관적 증거**를 항상 포함한다.

---

## 빠른 참고 치트시트

```bash
# 프로세스 확인
ps -ef | grep agent             # 프로세스 존재 확인
top -p $(pgrep agent-leak-app)  # 실시간 모니터링
top -H -p <PID>                 # 스레드 모니터링

# 로그 분석
tail -f app.log                                     # 실시간 로그
grep -i "critical\|error\|terminated" app.log       # 이상 로그 필터
grep "MEM:" monitor.log | awk '{print $1, $5}'      # 시간별 메모리

# 프로세스 종료
kill -15 <PID>   # 정상 종료 요청
kill -9 <PID>    # 강제 종료

# 환경변수
export MEMORY_LIMIT=512
source ~/.bash_profile           # 설정 파일 재로드

# 디렉터리/파일 확인
ls -la $AGENT_HOME               # 디렉터리 구조 확인
cat $AGENT_HOME/api_keys/secret.key  # 키 파일 내용 확인
```

---

> **학습 순서 추천**
>
> 1. 섹션 1~2 (OS, 프로세스 기초) → 개념 이해
> 2. 섹션 6~7 (리눅스 명령어, 환경변수) → 실습 준비
> 3. `agent-leak-app` 실행 및 `monitor.sh` 관찰
> 4. 섹션 3 (메모리 누수) → OOM 장애 분석
> 5. 섹션 4 (CPU) → CPU Spike 장애 분석
> 6. 섹션 5 (Deadlock) → Deadlock 장애 분석
> 7. 섹션 10 (트러블슈팅) → GitHub Issue 작성
