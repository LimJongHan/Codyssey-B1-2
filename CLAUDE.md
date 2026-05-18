# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Linux 시스템 장애(OOM, CPU Spike, Deadlock) 트러블슈팅 미션 레포지토리.
제공된 바이너리 `agent-app-leak`을 실행하며 장애를 재현·분석하고, GitHub Issue 형식의 리포트를 작성하는 것이 목표다.

## 핵심 파일

- `Misson.md` — 미션 요구사항 전체 (환경변수 조건, 리포트 템플릿, 케이스별 필수 증거 기준 포함)
- `study.md` — 비전공자용 개념 학습 가이드 (OS, 프로세스, 메모리, CPU, Deadlock, 리눅스 명령어)
- `agent-app-leak` — 실행 바이너리 (Python 기반, gitignore 처리됨)

## agent-app-leak 실행 조건

아래 환경변수가 모두 설정되어 있어야 부트 시퀀스를 통과한다.

```bash
export AGENT_HOME=<경로>
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=<로그 경로>
export MEMORY_LIMIT=256        # 50~512 MB
export CPU_MAX_OCCUPY=80       # 10~100 %
export MULTI_THREAD_ENABLE=true
```

`$AGENT_HOME/api_keys/secret.key` 파일에 `agent_api_key_test` 내용이 있어야 하며,
실행 계정은 반드시 root가 아닌 일반 사용자여야 한다.

## 장애 유형별 핵심

| 유형 | 조작 환경변수 | 종료 방식 | 식별 키워드 |
| --- | --- | --- | --- |
| OOM | `MEMORY_LIMIT` | SELF-TERMINATED | `MemoryGuard`, `Memory limit exceeded` |
| CPU Spike | `CPU_MAX_OCCUPY` | SIGTERM | `WATCHDOG`, `EMERGENCY ABORT` |
| Deadlock | `MULTI_THREAD_ENABLE=false`로 해제 | 종료 없음(무응답) | `WAITING`, `BLOCKED` |

## 리포트 작성 규칙

`reports/` 디렉터리에 케이스별 마크다운으로 작성한다.
각 리포트는 반드시 4개 섹션을 포함해야 한다: Description / Evidence & Logs / Root Cause Analysis / Workaround & Verification.

Before & After 비교 수치(MEM%, CPU%, 생존 시간)는 필수 증거다.

## 제약사항

- `agent-app-leak` 바이너리 디컴파일 및 리버스 엔지니어링 금지
- 사용 가능한 명령어: `monitor.sh`, `ps`, `top`, `htop`, `pstree`, `kill` 등 리눅스 표준 도구만 허용
