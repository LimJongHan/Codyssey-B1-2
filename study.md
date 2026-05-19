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
10. [네트워크 기초](#10-네트워크-기초)
11. [/proc 파일시스템](#11-proc-파일시스템)
12. [로그 시스템](#12-로그-시스템)
13. [트러블슈팅 사고법](#13-트러블슈팅-사고법)

---

## 1. 운영체제(OS) 기초

### 운영체제란 무엇인가?

컴퓨터는 크게 **하드웨어**(CPU, RAM, 디스크 등)와 **소프트웨어**(프로그램)로 나뉜다.
운영체제(OS)는 그 중간에서 하드웨어를 관리하고, 프로그램들이 하드웨어를 안전하게 사용할 수 있도록 중재한다.

```text
[사용자 프로그램]  ← 우리가 만드는 것
      ↕  (시스템 콜)
[운영체제 (OS)]    ← Linux, Windows, macOS
      ↕
[하드웨어]         ← CPU, RAM, 디스크, 네트워크 카드
```

### 커널(Kernel) vs 유저 공간(User Space)

OS는 크게 두 영역으로 나뉜다.

| 영역 | 설명 | 예시 |
| --- | --- | --- |
| **커널 공간** | OS 핵심. 하드웨어를 직접 제어. 오류 시 시스템 전체가 멈춤 | 메모리 관리, CPU 스케줄링, 드라이버 |
| **유저 공간** | 일반 프로그램이 실행되는 영역. 오류가 생겨도 해당 프로세스만 죽음 | python, nginx, agent-leak-app |

```text
[유저 공간]  agent-leak-app | python | bash
               ↕ 시스템 콜 (read, write, malloc...)
[커널 공간]  메모리 관리자 | 프로세스 스케줄러 | 파일시스템
               ↕
[하드웨어]   RAM | CPU | SSD
```

**왜 중요한가?** `agent-leak-app`은 유저 공간에서 실행된다. 아무리 메모리를 많이 써도 커널이 강제로 개입(OOM Killer)하거나, 앱 내부 정책(MemoryGuard)이 자신을 종료한다. 다른 프로세스에는 영향을 주지 않는다.

### 시스템 콜(System Call)

프로그램이 커널에게 "이 작업 해줘"라고 요청하는 창구.
파일 읽기, 메모리 할당, 네트워크 연결 등 **하드웨어가 필요한 모든 작업은 시스템 콜을 통한다**.

```text
프로그램: "파일 열어줘" → open() 시스템 콜 → 커널이 실제 디스크 접근
프로그램: "메모리 줘"   → malloc() → brk() 시스템 콜 → 커널이 RAM 할당
```

### 가상화와 컨테이너 (Docker)

물리 서버 1대 위에 여러 OS처럼 동작하는 환경을 만드는 기술.

| 기술 | 방식 | 특징 |
| --- | --- | --- |
| **VM (가상 머신)** | 하드웨어 전체를 가상화 | 무겁지만 완전한 격리 |
| **Docker (컨테이너)** | OS 커널을 공유, 프로세스만 격리 | 가볍고 빠름 |

이번 미션에서 Docker를 쓰는 이유: `agent-app-leak`은 Linux x86-64 ELF 바이너리다. macOS에서는 직접 실행 불가 → Docker로 Linux 환경을 만들어 실행한다.

```text
[macOS 호스트 (Apple Silicon ARM64)]
  └── [Docker 컨테이너: ubuntu 22.04 --platform linux/amd64]
        └── [agent-app-leak 프로세스 (x86-64 에뮬레이션)]
```

**Apple Silicon (M1/M2/M3/M4) 주의**: `--platform linux/amd64` 없이 실행하면 Rosetta가 x86-64 동적 링커를 찾지 못해 아래 에러가 발생한다.

```text
rosetta error: failed to open elf at /lib64/ld-linux-x86-64.so.2
```

올바른 실행 명령:

```bash
docker run -it --platform linux/amd64 --name codyssey-b12 \
  -v "$(pwd):/workspace" ubuntu:22.04 /bin/bash

uname -m   # x86_64 이 출력되어야 함
```

### 인터럽트(Interrupt)

하드웨어나 소프트웨어가 CPU에게 "지금 처리해야 할 일이 생겼어"라고 알리는 메커니즘.

- 키보드 입력, 타이머, 네트워크 패킷 수신 등이 인터럽트를 발생시킨다.
- CPU는 하던 일을 잠깐 멈추고 인터럽트를 처리한 뒤 다시 돌아온다.
- `Ctrl+C`를 누르면 SIGINT 인터럽트가 프로세스에 전달된다.

---

## 2. 프로세스와 스레드

### 프로그램 vs 프로세스

**프로그램**: 디스크에 저장된 실행 파일. 정지된 상태.
**프로세스**: 그 프로그램이 메모리에 올라와 실행 중인 상태.

```text
agent-app-leak (파일, 디스크)  →  실행  →  agent-app-leak (프로세스, RAM)
```

하나의 프로그램으로 여러 개의 프로세스를 동시에 실행할 수 있다.

### PID (Process ID)

프로세스가 시작되는 순간 OS가 부여하는 고유 번호. 1번부터 시작.

```text
PID 1    → systemd (리눅스 최초 프로세스, 모든 프로세스의 조상)
PID 1234 → agent-app-leak
PID 1235 → agent-app-leak의 자식 스레드
```

PID는 프로세스가 종료되면 반환되어 나중에 다른 프로세스에 재사용된다.

### 프로세스 생명주기

```text
생성(Created) → 준비(Ready) → 실행(Running) → 대기(Waiting) → 종료(Terminated)
                    ↑                                ↓
                    └────────────────────────────────┘
                    (I/O 완료, 이벤트 발생 등으로 다시 준비)
```

| 상태 | 설명 | top의 S 컬럼 |
| --- | --- | --- |
| Running | CPU를 실제로 사용 중 | `R` |
| Sleeping (대기) | 이벤트/I/O 기다리는 중 | `S` |
| Disk Sleep | 디스크 I/O를 강제로 기다리는 중 (중단 불가) | `D` |
| Zombie | 종료됐지만 부모가 회수 안 함 | `Z` |
| Stopped | 일시 정지 (Ctrl+Z) | `T` |

### 부모-자식 프로세스 관계

리눅스에서 모든 프로세스는 다른 프로세스가 낳는다.

```text
systemd(1)
  └── bash(100)
        └── agent-app-leak(1234)   ← bash가 부모, agent-app-leak이 자식
              ├── Thread-A(1235)
              └── Thread-B(1236)
```

- **fork()**: 현재 프로세스를 복제해서 자식 프로세스를 생성하는 시스템 콜
- **exec()**: 자식 프로세스가 다른 프로그램으로 교체되는 시스템 콜

### 좀비 프로세스 (Zombie)

자식 프로세스가 종료됐는데 **부모 프로세스가 자식의 종료 상태를 회수하지 않은** 상태.
PID는 살아있는 것처럼 보이지만 실제로 아무 작업도 하지 않는다.

```bash
ps -ef | grep Z   # 좀비 프로세스 확인
# STAT 컬럼에 Z가 있으면 좀비
```

> **Deadlock과 구분**: 좀비는 이미 종료됐지만 PID만 남음. Deadlock은 살아있지만 무응답.

### 고아 프로세스 (Orphan)

부모 프로세스가 먼저 죽어서 부모 없는 자식 프로세스.
리눅스는 자동으로 `systemd(PID 1)`가 양부모가 된다.

### 데몬 프로세스 (Daemon)

백그라운드에서 계속 실행되며 서비스를 제공하는 프로세스.
터미널과 분리되어 실행된다. 이름 끝에 `d`가 붙는 경우가 많다.

```bash
# 데몬 목록 예시
sshd    # SSH 서버
crond   # 크론 스케줄러
nginx   # 웹 서버
```

### 스레드(Thread)란?

프로세스 하나 안에서 동시에 여러 작업을 처리하기 위한 실행 단위.
프로세스가 **공장**이라면, 스레드는 그 안의 **작업자**다.

```text
[프로세스: agent-app-leak] ← 공장 (메모리 공간 1개)
  ├── Thread-A: 파일 업로드 처리   ← 작업자 1
  ├── Thread-B: API 요청 처리      ← 작업자 2
  └── Thread-C: 로그 기록          ← 작업자 3
```

**핵심**: 같은 프로세스 내 스레드들은 **메모리(힙)를 공유**한다. 이것이 Deadlock의 근본 원인.

### 프로세스 vs 스레드 비교

| 항목 | 프로세스 | 스레드 |
| --- | --- | --- |
| 메모리 공간 | 각자 독립적 | 같은 프로세스 내에서 공유 |
| 생성 비용 | 높음 (fork 필요) | 낮음 |
| 통신 방법 | IPC (파이프, 소켓 등) | 공유 메모리 직접 접근 |
| 오류 영향 | 해당 프로세스만 죽음 | 스레드 하나 오류 → 프로세스 전체 위험 |
| Deadlock 위험 | 낮음 | 높음 (메모리 공유로 인해) |

### 동기 vs 비동기

| 방식 | 설명 | 비유 |
| --- | --- | --- |
| **동기(Sync)** | 작업이 끝날 때까지 기다린 후 다음 진행 | 전화: 상대가 받을 때까지 기다림 |
| **비동기(Async)** | 작업을 요청하고 기다리지 않고 다음 진행, 완료 시 알림 받음 | 문자: 보내고 다른 일 하다가 답장 오면 확인 |

---

## 3. 메모리 구조와 메모리 누수

### RAM이란?

CPU가 직접 읽고 쓸 수 있는 고속 임시 저장소. 전원이 꺼지면 사라지는 **휘발성 메모리**.

```text
속도:  CPU 레지스터 > L1캐시 > L2캐시 > L3캐시 > RAM > SSD > HDD
용량:  CPU 레지스터 < L1캐시 < L2캐시 < L3캐시 < RAM < SSD < HDD
가격:  CPU 레지스터 > L1캐시 > L2캐시 > L3캐시 > RAM > SSD > HDD
```

CPU는 디스크에서 직접 읽지 않는다. 반드시 RAM → CPU 캐시 → CPU 레지스터 순으로 올린다.

### 가상 메모리 (Virtual Memory)

각 프로세스는 자신이 RAM 전체를 혼자 쓰는 것처럼 착각하게 해주는 기술.

```text
프로세스 A가 보는 세상:  주소 0x0000 ~ 0xFFFF (가상)
프로세스 B가 보는 세상:  주소 0x0000 ~ 0xFFFF (가상)
                  ↓ 커널의 페이지 테이블이 변환
실제 RAM:         A의 일부 + B의 일부 + 커널 = 물리 메모리
```

**장점**:
- 프로세스 간 메모리 격리 (A가 B의 메모리를 침범할 수 없음)
- RAM보다 큰 프로그램도 실행 가능 (일부만 RAM에 올리고 나머지는 디스크에)

### 페이지(Page)와 페이지 폴트(Page Fault)

가상 메모리는 **페이지(Page)**라는 고정 크기 블록(보통 4KB) 단위로 관리된다.

```text
페이지 폴트 발생 과정:
1. 프로세스가 메모리 주소에 접근
2. 해당 페이지가 RAM에 없음 → Page Fault 발생
3. 커널이 디스크(스왑)에서 해당 페이지를 RAM으로 불러옴
4. 프로세스 재개
```

Page Fault가 자주 발생하면 성능이 크게 떨어진다 (Disk I/O가 RAM보다 수백~수천 배 느리기 때문).

### 스왑(Swap) 메모리

RAM이 가득 찼을 때 디스크의 일부를 RAM처럼 쓰는 기술.

```text
RAM (8GB) 꽉 참 → 오래된 페이지를 Swap(디스크)으로 내보냄 → 새 데이터를 RAM에 올림
```

- **장점**: RAM 부족으로 인한 즉시 OOM을 지연시킴
- **단점**: 디스크는 RAM보다 수백 배 느림 → 시스템 전체가 느려짐 (Swap Storm)

```bash
# 스왑 사용량 확인
free -h
swapon --show
```

### 프로세스의 메모리 구조

```text
높은 주소 (예: 0xFFFFFFFF)
┌─────────────────┐
│   Stack         │  ← 함수 호출 시 자동 생성/삭제. 지역변수, 함수 인자
│   (아래로 성장) ↓│
│                 │
│   (위로 성장)   │
│   ↑  Heap       │  ← malloc/new로 직접 할당. 해제 안 하면 누수 발생
├─────────────────┤
│   BSS           │  ← 초기화 안 된 전역/정적 변수
├─────────────────┤
│   Data          │  ← 초기화된 전역/정적 변수
├─────────────────┤
│   Code (Text)   │  ← 실행할 프로그램 바이너리 코드 (읽기 전용)
낮은 주소 (예: 0x00000000)
```

### 힙(Heap)과 메모리 누수

**스택**: 함수가 끝나면 OS가 자동으로 회수. 프로그래머가 신경 쓸 필요 없음.
**힙**: 프로그래머가 직접 요청하고, **직접 해제해야** 한다. 해제 안 하면 누수.

```python
# Python 예시 - 메모리 누수 패턴
cache = {}
def process_request(request_id, data):
    cache[request_id] = data   # 추가만 하고 삭제 안 함
    # cache가 계속 커짐 → 힙 메모리 계속 증가 → 결국 OOM

# 해결법
def process_request_fixed(request_id, data):
    cache[request_id] = data
    if len(cache) > 1000:
        oldest = next(iter(cache))
        del cache[oldest]  # 오래된 항목 삭제
```

### 메모리 단편화 (Fragmentation)

메모리를 할당/해제를 반복하다 보면 **중간중간 사용 불가능한 빈 공간**이 생기는 현상.

```text
[사용중][비어있음 10MB][사용중][비어있음 10MB][사용중]
→ 총 20MB가 비어있지만 20MB 연속 공간이 없어서 20MB 요청 실패
```

### OOM(Out Of Memory)

메모리 누수가 계속되어 사용 가능한 메모리를 소진한 상태.

```text
[Linux OOM Killer]: OS가 가장 많은 메모리를 쓰는 프로세스를 선택해 강제 종료
[MemoryGuard]:      agent-app-leak 내부 정책이 MEMORY_LIMIT 초과 시 스스로 종료

dmesg 로그: "Out of memory: Killed process 1234 (agent-app-leak)"
```

### 이번 미션에서의 메모리 누수 흐름

```text
agent-app-leak 실행
     ↓
내부 로직이 힙에 데이터를 계속 쌓음 (해제 안 함)
     ↓
메모리 사용량 선형 증가 (monitor.sh로 관찰)
     ↓
MEMORY_LIMIT 도달 (예: 256MB)
     ↓
[CRITICAL] MemoryGuard 발동
     ↓
SELF-TERMINATED
```

### 메모리 관련 핵심 지표

| 지표 | 의미 | 확인 방법 |
| --- | --- | --- |
| **RSS** (Resident Set Size) | 실제 RAM에 올라온 크기 | `ps aux`, `top`의 RES |
| **VSZ** (Virtual Size) | 예약된 가상 메모리 크기 (RSS보다 항상 큼) | `ps aux`의 VSZ |
| **%MEM** | 전체 RAM 대비 사용 비율 | `top`, `ps aux` |
| **Shared** | 다른 프로세스와 공유하는 메모리 | `top`의 SHR |

```bash
# 상세 메모리 확인
cat /proc/<PID>/status | grep -i vm
# VmRSS: 실제 RSS
# VmSize: 가상 메모리 크기
# VmPeak: 최대 가상 메모리 (최고치)

# 시스템 전체 메모리 현황
free -h
cat /proc/meminfo
```

---

## 4. CPU와 스케줄링

### CPU란?

CPU(Central Processing Unit)는 컴퓨터의 두뇌. 코드를 실제로 **연산**하는 장치.

### 멀티코어와 하이퍼스레딩

**코어(Core)**: CPU 안의 실제 연산 유닛. 코어 수만큼 동시에 작업 처리 가능.
**하이퍼스레딩(Hyper-Threading)**: 코어 1개를 OS에게 2개처럼 보이게 하는 기술.

```bash
# CPU 정보 확인
lscpu
cat /proc/cpuinfo | grep "cpu cores"   # 물리 코어 수
cat /proc/cpuinfo | grep processor | wc -l  # 논리 CPU 수 (하이퍼스레딩 포함)
```

```text
예: 4코어 + 하이퍼스레딩 → OS에게는 8개 CPU처럼 보임
top에서 '1' 키 누르면 각 CPU별 사용률 확인 가능
```

### CPU 캐시 (L1, L2, L3)

RAM 접근은 느리다. CPU는 자주 쓰는 데이터를 고속 캐시에 저장한다.

```text
L1 캐시: 1~4MB, 접근 시간 ~1ns  (코어당)
L2 캐시: 4~16MB, 접근 시간 ~5ns (코어당)
L3 캐시: 16~64MB, 접근 시간 ~20ns (CPU 전체 공유)
RAM:     GB 단위, 접근 시간 ~100ns
```

**캐시 미스(Cache Miss)**: 찾는 데이터가 캐시에 없어 RAM까지 가야 하는 상황. 성능 저하 원인.

### CPU 사용률 읽는 법

`top`에서 보이는 CPU 사용률은 여러 항목으로 나뉜다.

```text
%Cpu(s): 5.2 us,  1.1 sy,  0.0 ni, 93.4 id,  0.0 wa,  0.0 hi,  0.3 si
```

| 항목 | 의미 | 높으면? |
| --- | --- | --- |
| `us` (user) | 유저 공간 코드 실행 | 앱 자체가 CPU 많이 사용 |
| `sy` (system) | 커널 코드 실행 (시스템 콜) | I/O, 시스템 작업 많음 |
| `id` (idle) | CPU가 놀고 있음 | 낮을수록 바쁨 |
| `wa` (iowait) | I/O 완료를 기다리는 시간 | 디스크/네트워크 병목 |
| `hi` (hardware interrupt) | 하드웨어 인터럽트 처리 | 네트워크/디스크 과부하 |

### 로드 에버리지 (Load Average) 상세 해석

```text
load average: 1.20, 0.85, 0.60
              1분   5분   15분
```

**해석 기준**: CPU 코어 수와 비교한다.

```text
4코어 시스템 기준:
  load average 2.0 → 50% 사용 (여유 있음)
  load average 4.0 → 100% 사용 (한계)
  load average 8.0 → 200% 사용 (과부하, 큐 대기 발생)
```

- 1분 > 15분: 갑자기 부하가 늘었음 → 최근에 무언가 시작됨
- 1분 < 15분: 부하가 줄어드는 추세 → 과부하 상황이 회복 중

```bash
# 코어 수 확인
nproc
# load average / nproc > 1 이면 과부하
```

### iowait란?

CPU가 할 일이 있는데 디스크/네트워크 I/O를 기다리느라 실행 못 하는 시간.
`iowait`가 높으면 CPU가 아니라 I/O가 병목이다.

```text
iowait 높음 = "CPU는 멀쩡한데 디스크/네트워크가 느려서 일을 못 함"
CPU 사용률 높음 = "CPU 자체가 바빠서 다른 프로세스가 밀려있음"
```

### CPU 스케줄링

CPU는 한 번에 하나의 작업만 처리한다 (코어 하나 기준).
여러 프로세스가 동시에 실행되는 것처럼 보이는 이유는 **매우 빠르게 번갈아가며 처리**하기 때문.

#### FCFS (First Come, First Served) - 선착순

```text
도착 순서: A → B → C
실행 순서: [AAAA 완료][BBBB 완료][CCCC 완료]
```

- **장점**: 단순, 구현 쉬움
- **단점**: A가 오래 걸리면 B, C가 오래 기다림 (Convoy Effect)
- **적합한 서비스**: 배치 처리, 파일 변환, 데이터 파이프라인

#### Round-Robin (라운드 로빈) - 시간 할당

```text
Time Quantum = 50ms
실행 순서: [A:50ms][B:50ms][C:50ms][A:50ms][B:50ms]...
```

- **장점**: 모든 프로세스가 공평하게 CPU를 사용, 응답성 좋음
- **단점**: 컨텍스트 스위칭 오버헤드, Time Quantum 설정이 중요
- **적합한 서비스**: 웹 서버, 대화형 시스템, 실시간 응답 필요한 서비스

#### Priority Scheduling - 우선순위

```text
우선순위: C(10) > B(5) > A(3)
실행 순서: [CCCC 완료][BBBB 완료][AAAA 완료]
```

- **장점**: 중요한 작업 우선 처리
- **단점**: 낮은 우선순위는 영원히 실행 못 할 수 있음 (Starvation)
- **해결**: Aging 기법 - 오래 기다릴수록 우선순위를 높여줌

#### 현대 리눅스: CFS (Completely Fair Scheduler)

리눅스 커널이 실제로 사용하는 스케줄러. 각 프로세스에 CPU 시간을 공평하게 분배한다.
Round-Robin + 가상 실행 시간 추적을 조합한 방식.

### 컨텍스트 스위칭 (Context Switching)

CPU가 한 프로세스에서 다른 프로세스로 전환할 때 발생하는 작업.

```text
1. 현재 프로세스 상태 저장 (레지스터, 프로그램 카운터, 스택 포인터 등)
2. 다음 프로세스 상태 복원
3. 새 프로세스 실행
```

- 저장/복원 과정에서 **시간이 소모** (오버헤드)
- 스레드 간 스위칭은 프로세스 간 스위칭보다 가볍다 (메모리 공유하므로)
- 스위칭이 너무 잦으면 실제 작업보다 스위칭 비용이 더 커질 수 있음

### nice 값과 우선순위 조정

리눅스에서 프로세스의 CPU 우선순위를 직접 조정할 수 있다.

```bash
# nice 값 범위: -20 (최고 우선순위) ~ 19 (최저 우선순위)
# 기본값: 0

nice -n 10 ./agent-app-leak    # 낮은 우선순위로 실행
renice -n 5 -p <PID>           # 실행 중인 프로세스 우선순위 변경

# top에서 PR(Priority)과 NI(Nice) 컬럼으로 확인
```

### CPU 과점유(CPU Spike)

특정 프로세스가 CPU를 과도하게 점유하여 다른 프로세스가 실행되지 못하는 상태.

```text
정상:  agent-app-leak CPU: 5~20%
과점유: agent-app-leak CPU: 95~100% → 다른 프로세스들이 응답 불가
```

이번 미션의 CPU 과점유 흐름:

```text
agent-app-leak 실행
     ↓
특정 시점에 무거운 연산 시작 → CPU 급상승
     ↓
CPU_MAX_OCCUPY 임계치 초과
     ↓
[Watchdog] 과점유 방지 정책 발동
     ↓
SIGTERM 전송 → 프로세스 종료
```

---

## 5. 교착상태(Deadlock)

### Deadlock이란?

두 개 이상의 스레드/프로세스가 **서로 상대방이 가진 자원을 기다리며 무한히 멈춰있는 상태**.
프로세스는 살아있지만 아무 진전이 없다.

### 뮤텍스(Mutex)와 세마포어(Semaphore)

Deadlock을 이해하려면 먼저 **락(Lock)** 개념을 알아야 한다.

**뮤텍스(Mutex, Mutual Exclusion)**:
공유 자원에 동시에 하나의 스레드만 접근하도록 잠그는 장치.

```text
화장실 열쇠와 같다.
- 스레드-A가 열쇠를 가져감 (Lock 획득)
- 스레드-B가 열쇠를 요청 → 없으니 대기
- 스레드-A가 열쇠를 반납 (Lock 해제)
- 스레드-B가 열쇠를 가져감
```

**세마포어(Semaphore)**:
N개의 스레드가 동시에 접근 가능하도록 허용하는 장치.

```text
주차장 입구와 같다. (총 3자리)
- 남은 자리 수: 3
- 차가 들어올 때마다 -1, 나갈 때마다 +1
- 0이 되면 새 차는 대기
```

### 경쟁 조건 (Race Condition)

여러 스레드가 공유 데이터에 동시에 접근할 때 실행 순서에 따라 결과가 달라지는 버그.

```text
공유 변수: count = 0

스레드-A: count = count + 1  ─┐
스레드-B: count = count + 1  ─┘ (동시 실행)

기대값: count = 2
실제값: count = 1 (한 쪽의 증가가 덮어씌워짐)
```

Race Condition을 막으려고 Mutex를 쓰는데, 잘못 쓰면 Deadlock이 발생한다.

### 식사하는 철학자 문제

가장 유명한 Deadlock 예시.

```text
         [철학자 1]
       /             \
  젓가락1           젓가락2
[철학자 5]         [철학자 2]
  젓가락5           젓가락3
       \             /
         [철학자 4]--젓가락4--[철학자 3]
```

- 식사하려면 왼쪽 + 오른쪽 젓가락 **둘 다** 필요
- 모든 철학자가 동시에 왼쪽 젓가락을 집음
- 모두 오른쪽 젓가락을 기다림
- 아무도 내려놓지 않음 → **영원히 아무도 식사 못 함** → Deadlock

### Deadlock의 4대 조건

아래 4가지가 **동시에** 성립해야 Deadlock 발생. 하나라도 깨면 방지 가능.

| 조건 | 설명 | 예시 |
| --- | --- | --- |
| **상호 배제** (Mutual Exclusion) | 자원은 한 번에 하나의 스레드만 사용 가능 | 젓가락은 한 명만 쥘 수 있음 |
| **점유 대기** (Hold and Wait) | 자원을 쥔 채로 다른 자원을 기다림 | 왼쪽 젓가락 쥔 채 오른쪽 기다림 |
| **비선점** (No Preemption) | 다른 스레드 자원을 강제로 빼앗을 수 없음 | 남의 젓가락 뺏을 수 없음 |
| **순환 대기** (Circular Wait) | A→B→C→A 형태로 순환 대기 | 1→2→3→4→5→1 순환 |

### 코드로 이해하는 Deadlock

```text
공유 자원: Lock-1, Lock-2

Thread-A의 작업 순서:
  1단계: Lock-1 획득 ✅
  2단계: Lock-2 획득 시도 → Lock-2는 Thread-B가 보유 중 → 무한 대기 🔴

Thread-B의 작업 순서:
  1단계: Lock-2 획득 ✅
  2단계: Lock-1 획득 시도 → Lock-1은 Thread-A가 보유 중 → 무한 대기 🔴

결과: 둘 다 영원히 대기 → 프로세스 전체 무응답
```

### Deadlock vs 유사 개념 구분

| 상황 | 설명 | CPU | MEM | 로그 | PID |
| --- | --- | --- | --- | --- | --- |
| **Deadlock** | 서로 기다리며 무응답 | 0% | 변화 없음 | 멈춤 | 존재 |
| **Livelock** | 계속 상태를 바꾸지만 진전 없음 | 높음 | 변화 있음 | 계속 출력 | 존재 |
| **Starvation** | 특정 스레드만 계속 무시됨 | 다른 스레드 높음 | 정상 | 부분적 | 존재 |
| **OOM 종료** | 메모리 초과로 강제 종료 | 0% | — | SELF-TERMINATED | 없음 |
| **Zombie** | 이미 종료됐지만 PID 남음 | 0% | 거의 없음 | 없음 | 존재 |

### Deadlock 예방 / 회피 / 탐지

| 전략 | 방법 | 예시 |
| --- | --- | --- |
| **예방** | 4대 조건 중 하나를 원천 차단 | 항상 Lock을 같은 순서로 획득 (순환 대기 제거) |
| **회피** | 자원 할당 전에 안전한지 검사 | 은행원 알고리즘 |
| **탐지 후 복구** | 주기적으로 탐지, 발생 시 스레드 종료 | Watchdog, Timeout |

이번 미션의 해결법: `MULTI_THREAD_ENABLE=false` → 스레드를 1개만 사용 → 순환 대기 원천 차단

### Deadlock 식별 명령어

```bash
# 1. PID 존재 확인 (살아있는지)
ps -ef | grep agent-app-leak

# 2. 스레드별 CPU 확인 (모두 0%인지)
top -H -p $(pgrep agent-app-leak)

# 3. 스레드 상태 확인
ps -L -p $(pgrep agent-app-leak) -o tid,stat,%cpu

# 4. 마지막 로그 확인
tail -30 app.log
grep -i "waiting\|blocked\|lock" app.log | tail -10
```

---

## 6. 리눅스 기본 지식

### 파일 시스템 구조

리눅스에서 **모든 것은 파일**이다. 디렉터리, 장치, 소켓, 파이프도 파일이다.

```text
/
├── bin/      ← 기본 명령어 (ls, cp, cat...)
├── sbin/     ← 시스템 관리 명령어 (root용)
├── etc/      ← 설정 파일들 (/etc/passwd, /etc/hosts...)
├── home/     ← 일반 사용자 홈 (/home/jonghan)
├── root/     ← root 사용자 홈
├── var/      ← 자주 변하는 파일 (로그: /var/log/, 임시: /var/tmp/)
├── tmp/      ← 임시 파일 (재부팅 시 삭제)
├── proc/     ← 가상 파일시스템. 커널/프로세스 정보를 파일 형태로 제공
├── sys/      ← 커널과 드라이버 정보
├── dev/      ← 장치 파일 (/dev/sda, /dev/null, /dev/random...)
└── usr/      ← 사용자 프로그램 (/usr/bin/, /usr/lib/...)
```

### 표준 입출력 (stdin, stdout, stderr)

리눅스의 모든 프로세스는 기본적으로 3개의 파일 디스크립터를 가진다.

| 번호 | 이름 | 기본 연결 | 설명 |
| --- | --- | --- | --- |
| 0 | stdin | 키보드 | 입력 |
| 1 | stdout | 터미널 화면 | 일반 출력 |
| 2 | stderr | 터미널 화면 | 에러 출력 |

```bash
# stdout을 파일로 저장
./agent-app-leak > app.log

# stderr도 같이 저장
./agent-app-leak > app.log 2>&1
# 2>&1 = "stderr(2)를 stdout(1)이 가는 곳으로 보내라"

# stdout은 화면에, stderr는 파일에
./agent-app-leak 2> error.log

# 출력을 완전히 버리기 (/dev/null은 블랙홀)
./agent-app-leak > /dev/null 2>&1
```

### 파이프(|)와 리다이렉션

**파이프**: 한 명령어의 stdout을 다음 명령어의 stdin으로 연결.

```bash
# ps의 출력을 grep에 넘김
ps -ef | grep agent

# 여러 단계 파이프
ps -ef | grep agent | grep -v grep | awk '{print $2}'
```

**리다이렉션**: 입출력 방향을 파일로 바꿈.

```bash
> file    # 덮어쓰기 (파일이 없으면 생성)
>> file   # 이어쓰기
< file    # 파일에서 읽기
```

### 파일 권한 상세

```bash
ls -la
# -rwxr-xr-x  1  jonghan  staff  7.6M  Jan 29  agent-app-leak
#  ↑↑↑↑↑↑↑↑↑
#  파일 유형 + 소유자권한(3) + 그룹권한(3) + 기타권한(3)
```

**파일 유형 문자**:

| 문자 | 의미 |
| --- | --- |
| `-` | 일반 파일 |
| `d` | 디렉터리 |
| `l` | 심볼릭 링크 |
| `s` | 소켓 |
| `p` | 파이프 |

**권한 숫자 표현**:

```text
r = 4, w = 2, x = 1

755 = rwxr-xr-x = 소유자(7=4+2+1) 그룹(5=4+1) 기타(5=4+1)
644 = rw-r--r-- = 소유자(6=4+2)   그룹(4=4)   기타(4=4)
600 = rw------- = 소유자(6=4+2)   그룹(0)     기타(0)
```

```bash
chmod 755 agent-app-leak   # 실행 가능하게
chmod 600 secret.key       # 소유자만 읽기/쓰기
chmod +x script.sh         # 실행 권한 추가

# 소유자 변경
chown agent-user:agent-user agent-app-leak
```

### 사용자와 그룹 관리

```bash
# 사용자 생성
useradd -m -s /bin/bash agent-user   # -m: 홈 디렉터리 생성, -s: 쉘 지정
passwd agent-user                     # 비밀번호 설정

# 사용자 전환
su - agent-user    # agent-user로 전환 (-는 환경변수도 같이 전환)
whoami             # 현재 사용자 확인

# sudo 권한 부여
usermod -aG sudo agent-user

# 사용자 정보 확인
id agent-user
cat /etc/passwd | grep agent-user
```

### 자주 쓰는 명령어

```bash
# 기본 탐색
pwd                     # 현재 위치
ls -la                  # 상세 목록 (숨김 포함)
cd ~                    # 홈 디렉터리
cd -                    # 이전 디렉터리

# 파일 조작
cp -r src/ dst/         # 디렉터리 복사
mv file.txt ../         # 상위 디렉터리로 이동
rm -rf dir/             # 디렉터리 삭제 (주의!)
ln -s target link       # 심볼릭 링크 생성

# 파일 내용 보기
cat file                # 전체 출력
less file               # 페이지 단위 보기 (q: 종료)
head -n 20 file         # 처음 20줄
tail -n 20 file         # 마지막 20줄
tail -f file            # 실시간 추가 내용 보기 (로그 모니터링)
tail -f file | grep -i error  # 실시간 필터링

# 검색
grep "keyword" file             # 키워드 찾기
grep -r "keyword" directory/    # 디렉터리 전체 재귀 검색
grep -i "keyword" file          # 대소문자 무시
grep -n "keyword" file          # 줄 번호 포함
grep -E "pattern1|pattern2" file # 정규식
grep -B3 -A3 "keyword" file     # 전후 3줄 포함

# 파일 찾기
find . -name "*.log"            # 현재 디렉터리 이하에서 .log 파일
find . -newer app.log           # app.log보다 최신 파일
find /home -user agent-user     # 특정 사용자 소유 파일

# 디스크 사용량
df -h                  # 파티션별 디스크 사용량
du -sh /home/          # 특정 디렉터리 크기
du -sh * | sort -rh    # 크기 순으로 정렬
```

### 패키지 관리 (apt)

```bash
apt-get update                    # 패키지 목록 갱신
apt-get install -y procps htop    # 패키지 설치
apt-get remove package-name       # 패키지 제거
apt-cache search keyword          # 패키지 검색
dpkg -l | grep procps             # 설치된 패키지 확인
```

### 프로세스 실행 방법

```bash
# 포그라운드 실행 (터미널 점유)
./agent-app-leak

# 백그라운드 실행 (터미널 반환)
./agent-app-leak &
echo $!    # 방금 실행한 프로세스의 PID 확인

# nohup: 터미널 종료 후에도 계속 실행
nohup ./agent-app-leak > app.log 2>&1 &

# 실행 중인 백그라운드 작업 확인
jobs

# 백그라운드 → 포그라운드
fg %1

# 포그라운드 → 백그라운드
Ctrl+Z    # 일시 정지
bg %1     # 백그라운드로 재개
```

---

## 7. 환경변수

### 환경변수란?

프로그램이 실행될 때 참조할 수 있는 **이름=값** 형태의 전역 설정값.
코드를 수정하지 않고 프로그램의 동작을 외부에서 바꿀 수 있다.

### 셸 변수 vs 환경변수

```bash
# 셸 변수: 현재 셸에서만 사용 가능, 자식 프로세스에 전달 안 됨
MY_VAR="hello"

# 환경변수: export로 내보내야 자식 프로세스(실행된 프로그램)에 전달됨
export MY_VAR="hello"

# 확인 방법
echo $MY_VAR          # 변수 값 출력
printenv MY_VAR       # 환경변수만 출력 (셸 변수는 안 보임)
env                   # 모든 환경변수 출력
set                   # 셸 변수 + 환경변수 모두 출력
```

### .bash_profile vs .bashrc 차이

| 파일 | 실행 시점 | 용도 |
| --- | --- | --- |
| `~/.bash_profile` | 로그인 시 1번 실행 | 환경변수, PATH 설정 |
| `~/.bashrc` | 새 터미널(비로그인 셸) 열 때마다 실행 | 별칭(alias), 함수 설정 |
| `~/.profile` | `.bash_profile` 없을 때 로그인 시 실행 | 범용 설정 |

**이번 미션**: `su - agent-user`로 로그인하면 `.bash_profile`이 읽힌다.

```bash
# ~/.bash_profile 편집
nano ~/.bash_profile

# 내용 예시
export AGENT_HOME=/home/agent-user/agent
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=$AGENT_HOME/logs
export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=80
export MULTI_THREAD_ENABLE=true

# 변경사항 즉시 적용 (현재 세션)
source ~/.bash_profile
# 또는
. ~/.bash_profile
```

### PATH 환경변수

명령어를 입력했을 때 어떤 디렉터리에서 찾을지 정의하는 변수.

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# which: 명령어가 어디 있는지 확인
which python3
# /usr/bin/python3

# PATH에 디렉터리 추가
export PATH=$PATH:/home/agent-user/scripts
```

### 이번 미션의 환경변수 전체 정리

| 환경변수 | 역할 | 허용 범위 |
| --- | --- | --- |
| `AGENT_HOME` | 앱의 기준 홈 디렉터리 | 존재하는 경로 |
| `AGENT_PORT` | 서버 포트 | 15034 (고정) |
| `AGENT_UPLOAD_DIR` | 업로드 파일 경로 | `$AGENT_HOME/upload_files` (존재 필수) |
| `AGENT_KEY_PATH` | API 키 디렉터리 경로 | `$AGENT_HOME/api_keys` (존재 필수) |
| `AGENT_LOG_DIR` | 로그 저장 경로 | 쓰기 권한 있는 경로 |
| `MEMORY_LIMIT` | 메모리 상한 (MB) | 50 ~ 512 |
| `CPU_MAX_OCCUPY` | CPU 점유율 상한 (%) | 10 ~ 100 |
| `MULTI_THREAD_ENABLE` | 멀티스레드 여부 | true/false, 1/0, yes/no |

### 환경변수 디버깅

```bash
# 모든 AGENT_ 관련 변수 확인
env | grep AGENT

# 특정 변수가 설정됐는지 확인
if [ -z "$AGENT_HOME" ]; then
    echo "AGENT_HOME이 설정되지 않았습니다"
fi

# 디렉터리 존재 여부 확인
ls -la $AGENT_HOME
ls -la $AGENT_UPLOAD_DIR
ls -la $AGENT_KEY_PATH
cat $AGENT_KEY_PATH/secret.key
```

---

## 8. 모니터링 명령어 실전

### ps - 프로세스 스냅샷

특정 시점의 프로세스 목록을 찍는다. (실시간 아님)

```bash
ps -ef                          # 전체 프로세스 (-e: all, -f: full format)
ps aux                          # BSD 스타일 (a: all, u: user format, x: no tty)
ps -ef | grep agent-app-leak    # 특정 프로세스 찾기
ps -ef | grep -v grep           # grep 자신 제외

# 스레드 포함 표시
ps -L -p <PID>                  # 특정 PID의 스레드 목록
ps -eLf | grep agent            # 모든 스레드 표시

# 원하는 컬럼만 출력
ps -p <PID> -o pid,ppid,%cpu,%mem,rss,vsz,stat,comm
```

**ps aux 컬럼 의미**:

```text
USER   PID  %CPU  %MEM   VSZ    RSS  TTY  STAT  START  TIME  COMMAND
user   1234  5.2   3.1  512000 25600  ?    S    10:00  1:23  agent-app-leak
```

| 컬럼 | 의미 |
| --- | --- |
| `%CPU` | CPU 사용률 |
| `%MEM` | 메모리 사용률 |
| `VSZ` | 가상 메모리 크기 (KB) |
| `RSS` | 실제 RAM 사용 크기 (KB) |
| `STAT` | 프로세스 상태 (S=sleep, R=run, Z=zombie, D=disk wait) |
| `TIME` | 누적 CPU 사용 시간 |

### top - 실시간 모니터링

```bash
top                       # 전체 프로세스 실시간
top -p $(pgrep agent-app-leak)  # 특정 PID만
top -H -p <PID>           # 스레드별 (Deadlock 확인)
top -d 1                  # 1초 갱신
top -b -n 5 > top.log     # 배치 모드, 5회 캡처해서 파일 저장
```

**top 내부 단축키**:

| 키 | 기능 |
| --- | --- |
| `M` | 메모리 사용량 순 정렬 |
| `P` | CPU 사용량 순 정렬 |
| `T` | CPU 누적 시간 순 정렬 |
| `1` | CPU 코어별 사용률 보기 |
| `H` | 스레드 보기 토글 |
| `k` | 특정 PID 종료 |
| `r` | nice 값 변경 |
| `u` | 특정 사용자만 보기 |
| `q` | 종료 |

### htop - top의 개선판

```bash
apt-get install -y htop
htop
htop -p <PID>   # 특정 PID만
```

- 마우스 클릭 지원
- 컬러 막대 그래프로 CPU/메모리 시각화
- F9: 시그널 전송, F6: 정렬, F5: 트리 보기

### free - 메모리 사용량

```bash
free -h          # 사람이 읽기 쉬운 단위 (GB, MB)
free -s 2        # 2초마다 갱신
watch -n 2 free -h  # watch로 주기적 갱신
```

```text
              total   used    free   shared  buff/cache  available
Mem:          7.7Gi   2.1Gi   3.2Gi   123Mi    2.4Gi      5.2Gi
Swap:         2.0Gi   0.0Gi   2.0Gi
```

| 항목 | 의미 |
| --- | --- |
| `total` | 전체 RAM |
| `used` | 사용 중 |
| `free` | 완전히 비어있는 메모리 |
| `buff/cache` | 파일 시스템 캐시 (필요시 해제 가능) |
| `available` | 실제로 사용 가능한 메모리 (free + 해제 가능한 cache) |

> **주의**: `free` 값이 0이어도 `available`이 크면 여유 있다.

### vmstat - 시스템 종합 현황

CPU, 메모리, 스왑, I/O를 한 번에 볼 수 있다.

```bash
vmstat 2 10    # 2초 간격으로 10회 출력
```

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 3200000  50000 2400000  0    0     0     1  150  200  5  1 93  1  0
```

| 컬럼 | 의미 |
| --- | --- |
| `r` | 실행 대기 중인 프로세스 수 (높으면 CPU 부족) |
| `b` | I/O 대기 중인 프로세스 수 |
| `swpd` | 사용 중인 스왑 크기 |
| `si/so` | 스왑 in/out (높으면 메모리 부족) |
| `bi/bo` | 디스크 블록 in/out |
| `us/sy/id/wa` | CPU 사용률 분류 |

### iostat - I/O 통계

```bash
apt-get install -y sysstat
iostat -x 2     # 2초 간격, 상세 I/O 통계
```

### df / du - 디스크 사용량

```bash
df -h           # 파티션별 사용량
df -h /         # 루트 파티션만

du -sh /workspace/   # 특정 디렉터리 총 크기
du -sh *             # 현재 디렉터리 항목별 크기
du -sh * | sort -rh  # 크기 순 정렬
```

### lsof - 열린 파일/소켓 확인

"List Open Files". 프로세스가 어떤 파일, 소켓, 포트를 사용 중인지 확인.

```bash
lsof -p <PID>                    # 특정 프로세스의 열린 파일
lsof -i :15034                   # 포트 15034를 사용 중인 프로세스
lsof -u agent-user               # 특정 사용자의 열린 파일
lsof | grep agent-app-leak       # 프로세스 이름으로 필터
```

### watch - 명령어 반복 실행

```bash
watch -n 1 'ps aux | grep agent-app-leak'  # 1초마다 갱신
watch -n 2 free -h                          # 2초마다 메모리 확인
watch -n 5 'cat /proc/<PID>/status | grep VmRSS'  # 5초마다 RSS 확인
```

### pstree - 프로세스 트리

```bash
pstree                  # 전체 트리
pstree -p               # PID 포함
pstree -p $(pgrep agent-app-leak)  # 특정 프로세스 중심
```

```text
# 출력 예시
agent-app-leak(1234)─┬─{Thread-A}(1235)
                     ├─{Thread-B}(1236)
                     └─{Thread-C}(1237)
```

### grep으로 로그 분석

```bash
# 기본 검색
grep "CRITICAL" app.log
grep -i "memory" app.log        # 대소문자 무시
grep -n "SELF-TERMINATED" app.log  # 줄 번호 포함
grep -c "ERROR" app.log         # 매칭 줄 수 세기

# 여러 키워드
grep -E "CRITICAL|ERROR|WARNING" app.log

# 전후 문맥
grep -B 5 -A 5 "SELF-TERMINATED" app.log  # 전 5줄, 후 5줄

# 역매칭 (해당 줄 제외)
grep -v "DEBUG" app.log

# 실시간 필터링
tail -f app.log | grep -i "critical\|error"

# monitor.log에서 MEM 수치만 추출
grep "MEM:" monitor.log | awk '{print $1, $2, $5}'
```

### awk - 텍스트 처리

```bash
# 특정 컬럼 추출 (공백 구분)
ps aux | awk '{print $1, $2, $3}'   # USER, PID, %CPU 출력

# 조건 필터
ps aux | awk '$3 > 50 {print $2, $3}'  # CPU 50% 이상인 PID

# monitor.log에서 시간과 MEM만
grep "MEM:" monitor.log | awk -F'[: %]' '{print $1, $8}'
```

---

## 9. 시그널(Signal)

### 시그널이란?

OS가 프로세스에게 보내는 **비동기 알림**. 프로세스는 시그널을 받으면 기본 동작을 수행하거나, 직접 핸들러를 등록해서 처리할 수 있다.

### 주요 시그널 목록

```bash
kill -l    # 모든 시그널 번호와 이름 출력
```

| 시그널 | 번호 | 기본 동작 | 처리 가능 | 설명 |
| --- | --- | --- | --- | --- |
| `SIGHUP` | 1 | 종료 | 가능 | 터미널 끊김, 설정 파일 재로드에 사용 |
| `SIGINT` | 2 | 종료 | 가능 | Ctrl+C 입력 |
| `SIGQUIT` | 3 | 코어덤프+종료 | 가능 | Ctrl+\ 입력 |
| `SIGKILL` | 9 | 강제 종료 | **불가** | OS가 직접 처리, 무조건 종료 |
| `SIGTERM` | 15 | 종료 | 가능 | 정상 종료 요청 (기본 kill) |
| `SIGSTOP` | 19 | 일시 정지 | **불가** | Ctrl+Z |
| `SIGCONT` | 18 | 재개 | 가능 | 일시 정지된 프로세스 재개 |
| `SIGUSR1` | 10 | 종료 | 가능 | 앱이 자유롭게 정의 가능 |
| `SIGUSR2` | 12 | 종료 | 가능 | 앱이 자유롭게 정의 가능 |

### kill 명령어

```bash
kill <PID>              # SIGTERM (정상 종료 요청)
kill -9 <PID>           # SIGKILL (강제 종료)
kill -15 <PID>          # SIGTERM (명시적)
kill -SIGTERM <PID>     # 이름으로도 가능

# 이름으로 종료
pkill agent-app-leak    # 프로세스 이름으로 종료
killall agent-app-leak  # 같은 이름의 모든 프로세스 종료

# pgrep: 이름으로 PID 찾기
pgrep agent-app-leak
kill $(pgrep agent-app-leak)
```

### SIGTERM vs SIGKILL

```text
SIGTERM (정상 종료, 번호 15):
  → 프로세스에게 "곧 종료하겠습니다" 알림
  → 프로세스가 받아서: 로그 저장, 파일 닫기, 정리 작업 후 종료
  → 프로세스가 무시할 수 있음 (핸들러에서 처리 안 하면)
  → Watchdog이 이걸 사용해 CPU Spike 시 종료

SIGKILL (강제 종료, 번호 9):
  → OS가 직접 프로세스를 즉시 종료
  → 프로세스 코드가 실행될 기회 없음 → 정리 작업 불가
  → 데이터 손실, 파일 손상 가능
  → 마지막 수단으로 사용
  → MemoryGuard가 내부적으로 이걸 사용 가능
```

**실전 팁**: 먼저 `kill -15`를 보내고, 5초 기다려도 종료 안 되면 `kill -9` 사용.

```bash
kill -15 $(pgrep agent-app-leak)
sleep 5
kill -9 $(pgrep agent-app-leak) 2>/dev/null   # 이미 종료됐으면 무시
```

### nohup - 터미널 종료 후에도 실행 유지

```bash
# 터미널을 닫아도 프로세스 계속 실행
nohup ./agent-app-leak > app.log 2>&1 &

# 이미 실행 중인 프로세스를 터미널에서 분리
disown %1
```

### trap - 시그널 핸들러 (쉘 스크립트)

```bash
#!/bin/bash
# 스크립트가 종료될 때(Ctrl+C 포함) 정리 작업 수행
trap 'echo "종료 중..."; rm -f /tmp/lock; exit' SIGTERM SIGINT

echo "실행 중..."
while true; do sleep 1; done
```

---

## 10. 네트워크 기초

### IP 주소

인터넷에서 각 기기를 구별하는 주소.

```text
IPv4: 192.168.1.100 (점으로 구분된 4개 숫자, 각 0~255)
IPv6: 2001:db8::1   (더 긴 주소, IPv4 부족 문제 해결)

특수 주소:
  127.0.0.1   = localhost (자기 자신)
  0.0.0.0     = 모든 네트워크 인터페이스 (전체 오픈)
  192.168.x.x = 사설 네트워크 (내부망)
```

### 포트(Port)

같은 컴퓨터에서 여러 프로그램이 네트워크를 사용할 때 구별하는 번호 (0~65535).

```text
IP 주소 = 아파트 동 번호
포트    = 호수

192.168.1.100:15034 = "192.168.1.100 컴퓨터의 15034번 포트"
```

**잘 알려진 포트**:

| 포트 | 서비스 |
| --- | --- |
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| **15034** | **agent-app-leak (이번 미션)** |

### TCP vs UDP

| 항목 | TCP | UDP |
| --- | --- | --- |
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | 높음 (전송 보장, 순서 보장) | 낮음 |
| 속도 | 상대적으로 느림 | 빠름 |
| 용도 | 웹, 파일 전송, SSH | 스트리밍, 게임, DNS |

agent-app-leak은 TCP 기반 서버. `AGENT_PORT=15034`로 TCP 소켓을 연다.

### 0.0.0.0 바인딩

서버가 네트워크를 열 때 어떤 IP에서 요청을 받을지 지정한다.

```text
0.0.0.0:15034  = 모든 네트워크 인터페이스에서 15034 포트 수신 (외부 접근 허용)
127.0.0.1:15034 = 자기 자신(localhost)에서만 수신 (외부 접근 차단)
```

### 방화벽 (Firewall)

네트워크 트래픽을 허용/차단하는 보안 시스템.

```bash
# 방화벽 상태 확인
ufw status           # UFW (Ubuntu Firewall)
iptables -L          # iptables 규칙 목록

# 포트 허용
ufw allow 15034/tcp

# Docker 컨테이너는 방화벽이 기본적으로 비활성화되어 있음
```

### 네트워크 상태 확인 명령어

```bash
# 열린 포트 확인
ss -tlnp                          # 모든 TCP 리스닝 포트
ss -tlnp | grep 15034             # 특정 포트 확인
netstat -tlnp                     # 구형 방식 (ss 권장)

# 연결 상태 확인
ss -s                             # 연결 통계 요약
ss -anp | grep agent-app-leak     # 특정 프로세스 연결

# 네트워크 인터페이스 확인
ip addr                           # IP 주소 목록
ifconfig                          # 구형 방식
```

---

## 11. /proc 파일시스템

### /proc이란?

리눅스의 특수 가상 파일시스템. 실제 파일이 없고, **커널이 실시간으로 내용을 생성**해서 보여준다.
프로세스 정보, 시스템 통계 등을 파일 형태로 읽을 수 있다.

```bash
ls /proc/
# 숫자 디렉터리들 = PID, 그 외 = 시스템 정보
```

### /proc/PID/ - 프로세스 정보

```bash
PID=$(pgrep agent-app-leak)

# 프로세스 상태 요약
cat /proc/$PID/status

# 출력 예시
Name:   agent-app-leak
State:  S (sleeping)
Pid:    1234
PPid:   100
Threads: 3           ← 스레드 수
VmPeak: 524288 kB    ← 최대 가상 메모리
VmSize: 512000 kB    ← 현재 가상 메모리
VmRSS:  26144 kB     ← 실제 RAM 사용량 (중요!)
VmSwap: 0 kB         ← 스왑 사용량
```

```bash
# 메모리 맵 (어떤 라이브러리가 어디 로드됐는지)
cat /proc/$PID/maps

# 열린 파일 디스크립터
ls -la /proc/$PID/fd

# 환경변수 확인
cat /proc/$PID/environ | tr '\0' '\n'

# 실행 명령어
cat /proc/$PID/cmdline | tr '\0' ' '

# CPU 스케줄링 정보
cat /proc/$PID/sched
```

### /proc/meminfo - 시스템 메모리 정보

```bash
cat /proc/meminfo

# 주요 항목
MemTotal:    8192000 kB   # 전체 RAM
MemFree:     2048000 kB   # 완전히 비어있는 메모리
MemAvailable: 5120000 kB  # 실제 사용 가능 (가장 중요)
Cached:      3072000 kB   # 파일 캐시
SwapTotal:   2097152 kB   # 전체 스왑
SwapFree:    2097152 kB   # 사용 가능한 스왑
```

### /proc/cpuinfo - CPU 정보

```bash
cat /proc/cpuinfo | grep -E "processor|model name|cpu cores|siblings"
```

### /proc/PID/io - I/O 통계

```bash
cat /proc/$PID/io
# rchar: 읽은 바이트
# wchar: 쓴 바이트
# read_bytes: 실제 디스크에서 읽은 바이트
# write_bytes: 실제 디스크에 쓴 바이트
```

---

## 12. 로그 시스템

### 로그가 중요한 이유

장애가 발생한 순간 화면은 이미 사라졌다. **로그만이 유일한 증거다.**
좋은 로그는 "언제, 무엇이, 왜" 발생했는지 알려준다.

### 로그 레벨 (심각도)

| 레벨 | 설명 | 예시 |
| --- | --- | --- |
| `DEBUG` | 개발용 상세 정보 | 함수 진입/퇴출, 변수 값 |
| `INFO` | 정상 동작 기록 | 서버 시작, 요청 처리 완료 |
| `WARNING` | 주의가 필요하지만 동작은 함 | 메모리 80% 도달 |
| `ERROR` | 오류 발생, 일부 기능 실패 | 파일 열기 실패 |
| `CRITICAL` | 심각한 오류, 시스템 영향 | 메모리 한계 초과 → 종료 |

agent-app-leak 로그에서 `[CRITICAL]`이 보이면 즉시 주목해야 한다.

### dmesg - 커널 메시지 로그

커널이 출력하는 메시지. OOM이 발생하면 여기에 기록된다.

```bash
dmesg | tail -50                    # 최근 커널 메시지
dmesg | grep -i "oom\|killed"       # OOM 관련 메시지
dmesg | grep -i "out of memory"     # 메모리 부족 메시지
dmesg --follow                      # 실시간 커널 메시지
dmesg -T                            # 타임스탬프를 사람이 읽기 쉽게
```

OOM 발생 시 dmesg 출력 예시:

```text
[123456.789] Out of memory: Killed process 1234 (agent-app-leak)
             total-vm:512000kB, anon-rss:256000kB, file-rss:0kB
```

### /var/log/ - 시스템 로그 디렉터리

```bash
ls /var/log/
# syslog    - 시스템 전체 로그
# auth.log  - 인증 관련 로그 (로그인, sudo 등)
# kern.log  - 커널 메시지
# dpkg.log  - 패키지 설치 기록

tail -f /var/log/syslog     # 시스템 로그 실시간 보기
```

### journalctl (systemd 로그)

```bash
journalctl -f                    # 실시간 전체 로그
journalctl -n 50                 # 최근 50줄
journalctl --since "10 min ago"  # 최근 10분
journalctl -p err                # 에러 레벨 이상만
```

### 로그 분석 실전 패턴

```bash
# 1. 로그 파일 크기 확인
ls -lh app.log monitor.log

# 2. 로그가 언제 시작됐는지 확인
head -1 app.log

# 3. 로그가 언제 멈췄는지 확인
tail -1 app.log

# 4. 에러/경고 개수 파악
grep -c "CRITICAL\|ERROR" app.log
grep -c "WARNING" app.log

# 5. 시간대별 MEM 수치 추출
grep "MEM:" monitor.log | awk '{print $1, $2, $5}' | head -30

# 6. 종료 직전 30줄 + 에러 로그 함께 확인
tail -30 app.log
grep -E "CRITICAL|ERROR|TERMINATED|WATCHDOG|BLOCKED" app.log

# 7. 두 로그 파일의 시간대 맞추기
grep "14:05" monitor.log    # 14:05 시점 관제 데이터
grep "14:05" app.log        # 같은 시각 앱 로그
```

### 좋은 로그 수집 습관

```bash
# 앱 실행 시 로그 파일에 저장 (이어쓰기)
./agent-app-leak 2>&1 | tee -a app.log
# tee: 화면에도 출력하면서 파일에도 저장
# -a: 이어쓰기 (append)

# 타임스탬프 추가
./agent-app-leak 2>&1 | while IFS= read -r line; do
    echo "$(date '+%Y-%m-%d %H:%M:%S') $line"
done | tee -a app.log
```

---

## 13. 트러블슈팅 사고법

### 5단계 분석 프레임워크

```text
1. 관측 (Observe)
   "무엇이 이상한가?" 현상을 정확히 기술
   → "agent-app-leak이 약 10분 후 갑자기 종료됨"

2. 증거 수집 (Collect Evidence)
   "어떤 데이터가 있는가?" 로그, 수치, 출력 결과 수집
   → monitor.log, app.log, ps/top 출력 수집

3. 원인 분석 (Root Cause Analysis)
   "왜 발생했는가?" 증거를 근거로 기술적 원인 도출
   → "MEM이 선형 증가 → MEMORY_LIMIT 도달 → MemoryGuard 발동"

4. 조치 (Workaround)
   "어떻게 해결하는가?" 환경변수 조정, 설정 변경
   → "MEMORY_LIMIT을 256에서 512로 증가"

5. 검증 (Verify)
   "효과가 있었는가?" Before/After 비교
   → "Before: 10분 생존 / After: 30분 이상 생존"
```

### 5 Whys 기법

근본 원인을 찾기 위해 "왜?"를 5번 반복한다.

```text
현상: 프로세스가 종료됐다
  Why 1: 왜 종료됐는가? → MemoryGuard가 강제 종료했다
  Why 2: 왜 MemoryGuard가 발동했는가? → MEMORY_LIMIT을 초과했다
  Why 3: 왜 MEMORY_LIMIT을 초과했는가? → 메모리가 계속 증가했다
  Why 4: 왜 메모리가 증가했는가? → 힙에 데이터를 추가만 하고 해제하지 않는다
  Why 5: 왜 해제하지 않는가? → 코드에 메모리 해제 로직이 없다 (Memory Leak)

근본 원인 = "코드 내 Memory Leak"
임시 조치 = MEMORY_LIMIT 증가 (시간을 버는 것, 근본 해결 아님)
근본 해결 = 코드 수정 (메모리 해제 로직 추가)
```

### 장애 유형 구분 플로우차트

```text
프로세스가 이상하다
       ↓
PID가 있는가? (ps -ef | grep agent)
  ├── 없음 → 프로세스 종료됨
  │     ↓
  │   로그 확인
  │     ├── "SELF-TERMINATED", "Memory limit exceeded" → OOM
  │     └── "WATCHDOG", "SIGTERM", "CPU" → CPU Spike
  │
  └── 있음 → 프로세스 살아있음
        ↓
      로그가 계속 출력되는가? (tail -f app.log)
        ├── 계속 출력 → 정상 또는 다른 문제
        └── 멈춤 → Deadlock 의심
              ↓
            CPU 사용률 확인 (top -H -p <PID>)
              ├── 0% 또는 매우 낮음 → Deadlock 확정
              └── 높음 → 무한 루프 등 다른 문제
```

### 케이스별 1차 체크리스트

#### OOM 의심 시

```bash
# 프로세스 종료 확인
ps -ef | grep agent-app-leak         # PID 없으면 종료됨

# 앱 로그에서 메모리 관련 키워드
grep -i "memory\|terminated\|memoryguard" app.log

# 관제 로그에서 MEM 추이
grep "MEM:" monitor.log

# 커널 OOM 확인
dmesg | grep -i "out of memory\|killed process"

# 시스템 메모리 현황
free -h
cat /proc/meminfo | grep -E "MemAvailable|SwapFree"
```

#### CPU Spike 의심 시

```bash
# CPU 사용률 확인
top -p $(pgrep agent-app-leak)

# 앱 로그에서 Watchdog 키워드
grep -i "watchdog\|sigterm\|abort\|cpu" app.log

# 관제 로그에서 CPU 급상승 구간
grep "CPU:" monitor.log

# 프로세스별 CPU 스냅샷
ps aux --sort=-%cpu | head -10
```

#### Deadlock 의심 시

```bash
# 1. PID 존재 확인
ps -ef | grep agent-app-leak

# 2. 스레드별 CPU 확인 (모두 0%인지)
top -H -p $(pgrep agent-app-leak)

# 3. 로그 멈춤 확인
tail -f app.log            # 새 로그가 안 나오면 Deadlock

# 4. 마지막 로그 키워드
tail -30 app.log
grep -i "waiting\|blocked\|lock" app.log | tail -10

# 5. 스레드 상태
ps -L -p $(pgrep agent-app-leak) -o tid,stat,%cpu
```

### 타임라인 작성

장애 분석 시 시간대별로 무슨 일이 있었는지 정리한다.

```text
[타임라인 예시]
14:00:00 - agent-app-leak 실행 (MEMORY_LIMIT=256)
14:00:00 - monitor.sh 시작 → MEM: 5.1%
14:03:00 - monitor.log: MEM 35.4%
14:06:00 - monitor.log: MEM 68.2%
14:09:00 - monitor.log: MEM 89.5%
14:10:30 - app.log: [CRITICAL] MemoryGuard Memory limit exceeded
14:10:30 - app.log: SELF-TERMINATED
14:10:31 - monitor.log: PROCESS:NOT_FOUND
```

### GitHub Issue 작성 원칙

좋은 기술 리포트는 **재현 가능**하고 **객관적 증거**로 뒷받침돼야 한다.

```text
나쁜 예:
"메모리가 많이 쓰여서 종료되었다."
→ 얼마나? 언제? 왜? 증거는?

좋은 예:
"MEMORY_LIMIT=256 설정 하에 14:00:00에 agent-app-leak을 실행한 뒤,
monitor.log에서 MEM이 선형으로 증가하여 14:10:00에 96.8%에 도달하였으며,
app.log에서 [CRITICAL] [MemoryGuard] Memory limit exceeded (256MB >= 256MB) 로그가
출력된 직후 >>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<< 메시지와 함께
프로세스가 종료되었다 (PID: 1234, 생존 시간: 약 10분)."
```

**작성 체크리스트**:

- [ ] 발생 시각 또는 조건 명시
- [ ] 환경변수 설정값 명시
- [ ] 로그 원문 발췌 (수정 없이)
- [ ] monitor.log 수치 포함
- [ ] ps/top 등 시스템 도구 출력 포함
- [ ] Before & After 수치 비교

---

## 빠른 참고 치트시트

```bash
# ── 프로세스 확인 ──────────────────────────────
ps -ef | grep agent               # 프로세스 존재 확인
pgrep agent-app-leak              # PID만 출력
top -p $(pgrep agent-app-leak)    # 실시간 모니터링
top -H -p $(pgrep agent-app-leak) # 스레드별 CPU (Deadlock 확인)
ps -L -p $(pgrep agent-app-leak) -o tid,stat,%cpu  # 스레드 상태

# ── 메모리 확인 ────────────────────────────────
free -h                           # 시스템 메모리
cat /proc/$(pgrep agent-app-leak)/status | grep Vm  # 프로세스 메모리

# ── 로그 분석 ──────────────────────────────────
tail -f app.log                   # 실시간 로그
tail -f app.log | grep -i "critical\|error\|terminated"
grep "MEM:" monitor.log           # MEM 수치 추이
grep "CPU:" monitor.log           # CPU 수치 추이
dmesg | grep -i "oom\|killed"     # 커널 OOM 로그

# ── 프로세스 종료 ──────────────────────────────
kill -15 $(pgrep agent-app-leak)  # 정상 종료 요청
kill -9 $(pgrep agent-app-leak)   # 강제 종료

# ── 환경변수 ───────────────────────────────────
env | grep AGENT                  # AGENT 관련 변수 확인
export MEMORY_LIMIT=512
export CPU_MAX_OCCUPY=90
export MULTI_THREAD_ENABLE=false
source ~/.bash_profile             # 설정 재로드

# ── 디렉터리/파일 확인 ─────────────────────────
ls -la $AGENT_HOME
cat $AGENT_KEY_PATH/secret.key
ls -la $AGENT_UPLOAD_DIR
ls -la $AGENT_LOG_DIR

# ── 네트워크/소켓 ──────────────────────────────
ss -tlnp | grep 15034             # 포트 바인딩 확인
lsof -i :15034                    # 포트 사용 프로세스
```

---

> **학습 순서 추천**
>
> 1. 섹션 1~2 (OS, 프로세스 기초) → 전체 개념 틀 잡기
> 2. 섹션 6 (리눅스 기본 명령어) → 손에 익히기
> 3. 섹션 7 (환경변수) → 앱 실행 준비
> 4. 섹션 8 (모니터링 명령어) → ps, top, grep 실습
> 5. `agent-app-leak` 실행 + `monitor.sh` 관찰
> 6. 섹션 3 (메모리) → OOM 장애 분석
> 7. 섹션 4 (CPU) → CPU Spike 장애 분석
> 8. 섹션 5 (Deadlock) → Deadlock 장애 분석
> 9. 섹션 9 (시그널) → kill, SIGTERM 이해
> 10. 섹션 10~12 (네트워크, /proc, 로그) → 심화 분석
> 11. 섹션 13 (트러블슈팅) → 리포트 작성
