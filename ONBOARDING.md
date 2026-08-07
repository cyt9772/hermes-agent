# Hermes Agent 온보딩 문서

새 팀원이 코드베이스를 빠르게 파악할 수 있도록 정리한 문서입니다.

---

## 1. 프로젝트 개요

Hermes Agent는 하나의 Python 코어(`AIAgent`)를 CLI, 메신저(Telegram/Discord/Slack/WhatsApp 등), TUI, REST API 등 여러 인터페이스에서 공유해서 쓰는 **멀티 서페이스 AI 에이전트 런타임**입니다. 대화 중 스킬을 스스로 만들고 개선하며, 과거 대화를 검색하고, 세션 간 사용자 모델을 축적하는 **자가 개선(self-improving) 루프**가 핵심 차별점입니다. `$5 VPS`부터 서버리스 샌드박스까지 어디서든 상시 구동되며, 사용자는 원하는 채널에서 접속해 대화합니다.

---

## 2. 디렉토리 구조와 각 폴더 역할

```
run_agent.py       AIAgent 클래스 정의 — 에이전트의 심장부 (대화 루프 forwarder)
cli.py             HermesCLI — 인터랙티브 TUI 진입점
model_tools.py      tools/registry.py 위의 얇은 오케스트레이션 레이어
toolsets.py         툴을 묶은 프리셋 정의 (hermes-cli, hermes-telegram 등)
hermes_state.py      SQLite 기반 세션 저장소 (FTS5 전문 검색)

agent/              에이전트 내부 로직 (run_agent.py에서 분리된 모듈들)
  ├─ conversation_loop.py   실제 대화 루프 구현체 (LLM 호출 ↔ 툴 실행 반복)
  ├─ prompt_builder.py      시스템 프롬프트 조립 (정체성, 스킬 인덱스, 컨텍스트 파일, 메모리)
  ├─ context_compressor.py  컨텍스트 한도 근접 시 자동 요약
  └─ curator.py             스킬 생명주기 관리 (자가 개선 루프의 핵심)

hermes_cli/         CLI 명령어 구현 (config, setup wizard, auth/OAuth, doctor, skills hub)
tools/              자가 등록(self-registering) 방식의 툴 구현체
  ├─ registry.py        모든 툴의 중앙 등록 테이블 (스키마 + 핸들러)
  ├─ approval.py         위험 명령어 탐지 및 세션별 승인 게이트
  ├─ delegate_tool.py     서브에이전트 생성/병렬 작업 위임
  └─ environments/        터미널 실행 백엔드 (local, Docker, SSH, Singularity, Modal 등)

gateway/            메시징 게이트웨이
  ├─ run.py              GatewayRunner — 플랫폼 생명주기, 메시지 라우팅, cron 배포
  ├─ session.py           세션 저장소, 컨텍스트 프롬프트, 리셋 정책
  └─ platforms/           플랫폼별 어댑터 (telegram, discord, slack, whatsapp 등)

tui_gateway/        대시보드/TUI 백엔드 서버 (세션 실행, compute host 연동)
scripts/            설치 스크립트 (install.sh, install.ps1) + 테스트 러너
skills/             번들 스킬 (설치 시 ~/.hermes/skills/ 로 복사)
optional-skills/    커뮤니티/공식 옵션 스킬 (허브를 통해 옵트인)
plugins/            model-providers, memory-providers 등 확장 플러그인
apps/               데스크톱(Electron) 앱
tests/, tests-js/   Python / JS·TS 테스트 스위트
website/            Docusaurus 문서 사이트
cli-config.yaml.example   설정 예시 (설치 시 ~/.hermes/config.yaml 로 복사)
AGENTS.md           AI 코딩 어시스턴트용 개발 가이드 (필독)
```

---

## 3. 핵심 파일 5개와 역할

| # | 파일 | 역할 |
|---|------|------|
| 1 | `run_agent.py` | `AIAgent` 클래스 정의. 생성자에서 provider/모델/툴셋 등 설정을 받고, `run_conversation()`/`chat()`으로 대화를 실행하는 **모든 진입점의 최종 수렴 지점**. |
| 2 | `agent/conversation_loop.py` | `run_conversation()`의 실제 구현체. "LLM 호출 → tool_calls 있으면 실행 → 없으면 종료" 루프가 여기 있음 (라인 1415 부근 `while` 루프). |
| 3 | `tools/registry.py` | 모든 툴이 모듈 임포트 시점에 `register()`로 스스로 등록하는 중앙 레지스트리. 에이전트가 "무엇을 할 수 있는지"를 결정. |
| 4 | `hermes_state.py` | SQLite + FTS5 기반 세션 저장소. 대화 기록, 메타데이터, 검색을 담당해 세션 간 연속성을 보장. |
| 5 | `gateway/run.py` | `GatewayRunner` — Telegram/Discord/Slack/WhatsApp 등 여러 플랫폼 어댑터를 구동하고 메시지를 세션에 라우팅하는 오케스트레이터. |

---

## 4. 주요 코드 흐름

### 4-1. 대화 실행 흐름 (공통 경로)

```
사용자 입력 (CLI / 메신저 / TUI / REST API)
        │
        ▼
AIAgent.run_conversation()   ← run_agent.py:7759 (forwarder)
        │  (session/turn 부기, relay lease 획득 등 처리 후 위임)
        ▼
agent.conversation_loop.run_conversation()   ← 실제 루프
        │
        ├─ 시스템 프롬프트 조립 (agent/prompt_builder.py)
        ├─ while (반복 한도 이내):
        │     ├─ LLM 호출 (client.chat.completions.create)
        │     ├─ tool_calls 있음 → model_tools.handle_function_call()
        │     │        → tools/registry.py 디스패치 → 실제 툴 실행
        │     │        → 결과를 메시지에 추가, 반복 계속
        │     └─ tool_calls 없음 → 최종 응답으로 루프 종료
        ▼
결과 dict 반환 (final_response, messages, completed, api_calls)
```

이 루프에 도달하는 진입점은 여러 개지만 (인터랙티브 CLI, one-shot CLI, 메시징 게이트웨이, TUI, REST API, 서브에이전트 위임, 배치 실행, 큐레이터/백그라운드 리뷰, ACP), **실제 대화 루프 구현은 `agent/conversation_loop.py` 단 하나**입니다.

### 4-2. `python run_agent.py` 직접 실행 시 흐름

1. 모듈 임포트 시점에 `model_tools`/`tools/registry.py`를 통해 전체 툴 카탈로그가 로드됨 (각 툴이 자가 등록).
2. `if __name__ == "__main__": fire.Fire(main)` → CLI 인자를 파싱해 `main()` 호출.
3. `main()`이 `AIAgent(...)`를 생성하고 `agent.run_conversation(user_query)` 호출.
4. 위 4-1의 루프 실행 후 결과 요약을 출력.

### 4-3. 툴 추가 방식

`tools/<name>_tool.py` 파일에서 모듈 레벨로 `registry.register(...)`를 호출하면 `model_tools.py`가 임포트 시 자동으로 수집합니다. 새 툴을 추가할 때 `model_tools.py`나 `run_agent.py`를 건드릴 필요가 없습니다.

---

## 5. 개발 시작하기

### 5-1. 사전 요구사항

| 항목 | 비고 |
|---|---|
| Git | `git-lfs` 확장 포함 |
| Python 3.11–3.13 | 없으면 uv가 자동 설치 |
| uv | 빠른 Python 패키지 매니저 ([설치 가이드](https://docs.astral.sh/uv/)) |
| Node.js 20+ | 선택 — 브라우저 툴/WhatsApp 브릿지에 필요 |

### 5-2. 설치 (표준 설치 스크립트 방식, 권장)

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
cd "${HERMES_HOME:-$HOME/.hermes}/hermes-agent"

# 개발/테스트용 extras 추가 설치
uv pip install -e ".[all,dev]"

# 선택: 브라우저 툴 / 문서 사이트 의존성
npm install
```

### 5-3. 개발 환경 설정

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env

# 최소한 LLM provider 키 하나는 필요
echo "OPENROUTER_API_KEY=***" >> ~/.hermes/.env
```

### 5-4. 실행

```bash
hermes doctor        # 진단
hermes chat -q "Hello"   # 간단한 대화 테스트
```

> ⚠️ **주의**: 소스 트리 안에 venv를 만들지 마세요. 에이전트가 자기 자신의 체크아웃에 대해 `rm -rf venv` 같은 상대경로 명령을 실행하면 실행 중인 런타임이 삭제될 수 있습니다. 수동 클론 방식을 쓸 경우 venv는 반드시 트리 **바깥**(`~/.hermes/venvs/hermes-dev` 등)에 생성하세요.

### 5-5. 테스트

```bash
# 권장 — CI와 동일한 방식 (env -i로 격리, 파일별 서브프로세스 병렬 실행)
scripts/run_tests.sh

# 대안 (venv 활성화 후)
pytest tests/ -v
```

### 5-6. 브랜치/PR 흐름

```bash
git checkout -b fix/description
# ... 작업 ...
scripts/run_tests.sh
```

자세한 기여 규칙, 스타일 가이드, 아키텍처 상세는 `AGENTS.md`와 `CONTRIBUTING.md`를 참고하세요.
