# B1-2 리눅스 프로세스 및 시스템 리소스 트러블슈팅

| 항목 | 내용 |
| --- | --- |
| 분야 | AI/SW 기초 |
| 구분 | Linux와 OS |
| 학습시간 | 40시간 |

## 미션 개요

`agent-leak-app`을 실제 운영 환경처럼 실행하면서 발생하는 세 가지 시스템 장애를 분석하고,
GitHub Issue 형태의 기술 리포트로 기록한다.

| 장애 유형 | 핵심 증상 | 관련 환경변수 |
| --- | --- | --- |
| **OOM (메모리 누수)** | 메모리 선형 증가 후 SELF-TERMINATED | `MEMORY_LIMIT` |
| **CPU Spike (과점유)** | CPU 급상승 후 Watchdog SIGTERM | `CPU_MAX_OCCUPY` |
| **Deadlock (교착상태)** | PID 존재하나 무응답, 로그 멈춤 | `MULTI_THREAD_ENABLE` |

## 디렉터리 구조

```text
B1-2/
├── README.md       - 프로젝트 개요 (이 파일)
├── Misson.md       - 미션 요구사항 전체 정리
├── study.md        - 비전공자를 위한 개념 학습 가이드
└── reports/        - 장애 분석 리포트 (작성 예정)
    ├── oom.md
    ├── cpu.md
    └── deadlock.md
```

## 제출 결과물

- [ ] OOM 장애 분석 리포트
- [ ] CPU Spike 장애 분석 리포트
- [ ] Deadlock 장애 분석 리포트
- [ ] (선택) 스케줄링 알고리즘 추론 리포트

## 사전 환경 설정

`agent-leak-app` 실행 전 아래 환경변수를 `.bash_profile`에 설정해야 한다.

```bash
export AGENT_HOME=<경로>
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=<로그 경로>
export MEMORY_LIMIT=256        # 50~512 (MB)
export CPU_MAX_OCCUPY=80       # 10~100 (%)
export MULTI_THREAD_ENABLE=true
```

`$AGENT_HOME/api_keys/secret.key` 파일에 `agent_api_key_test` 내용이 있어야 한다.

## 학습 자료

개념이 생소한 경우 [`study.md`](./study.md)를 먼저 읽는 것을 권장한다.

커버 범위: OS 기초 / 프로세스·스레드 / 메모리 구조 / CPU 스케줄링 / Deadlock /
리눅스 명령어 / 환경변수 / 모니터링 도구(ps, top, htop) / 시그널 / 트러블슈팅 사고법
