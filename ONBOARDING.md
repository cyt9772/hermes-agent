# Hermes Agent 온보딩 문서

새 팀원이 코드베이스를 빠르게 파악할 수 있도록 정리한 문서입니다.

---

## 1. 프로젝트 개요

Hermes Agent는 하나의 Python 코어(`AIAgent`)를 CLI, 메신저(Telegram/Discord/Slack/WhatsApp 등), TUI, 데스크톱 앱, REST API 등 여러 인터페이스에서 공유해서 쓰는 **멀티 서페이스 AI 에이전트 런타임**입니다. 대화 중 스킬을 스스로 만들고 개선하며, 과거 대화를 검색하고, 세션 간 사용자 모델을 축적하는 **자가 개선(self-improving) 루프**가 핵심 차별점입니다. `$5 VPS`부터 서버리스 샌드박스까지 어디서든 상시 구동되며, 사용자는 원하는 채널에서 접속해 대화합니다.

해결하는 문제는 세 가지입니다.
- **세션 간 망각**: 메모리, 스킬, 세션 검색으로 해결합니다.
- **노트북 종속**: 원격 서버에서 실행하고 메신저로 접속합니다.
- **모델 종속**: `hermes model`로 프로바이더를 자유롭게 전환합니다.

---

## 2. 디렉토리 구조와 각 폴더 역할

```
run_agent.py        AIAgent 클래스 — 모든 인터페이스가 공유하는 에이전트 본체 (초기화·루프는 agent/로 위임)
cli.py              HermesCLI — prompt_toolkit 기반 클래식 대화형 CLI
model_tools.py      툴 스키마 생성(get_tool_definitions) + 툴 호출 디스패치(handle_function_call)
toolsets.py         툴 묶음 정의 (_HERMES_CORE_TOOLS, 플랫폼별 toolset)
hermes_state.py     SQLite 기반 세션 저장소 SessionDB (FTS5 전문 검색)
hermes_constants.py get_hermes_home() / display_hermes_home() — 프로필 인식 경로

agent/              에이전트 내부 로직 (run_agent.py에서 분리된 모듈들)
  ├─ agent_init.py          AIAgent.__init__ 본체 (클라이언트, 툴, 세션, 메모리, 압축기 초기화)
  ├─ conversation_loop.py   실제 대화 루프 (LLM 호출 ↔ 툴 실행 반복, 재시도/fallback)
  ├─ turn_context.py        턴 준비 (시스템 프롬프트, 사용자 메시지, preflight 압축, 훅)
  ├─ turn_finalizer.py      턴 마무리 (세션 저장, 정리, post 훅, 메모리 sync)
  ├─ tool_executor.py       툴 호출 병렬/순차 실행
  ├─ prompt_builder.py      시스템 프롬프트 조립 (정체성, 스킬 인덱스, 컨텍스트 파일, 메모리)
  ├─ context_compressor.py  컨텍스트 한도 근접 시 자동 요약
  ├─ memory_manager.py      메모리 프로바이더 오케스트레이션
  └─ curator.py             스킬 생명주기 관리 (자가 개선 루프의 핵심)

hermes_cli/         `hermes` 명령 구현 (main.py 진입점, config, setup wizard, auth/OAuth, doctor, skills hub, 스킨)
tools/              자가 등록(self-registering) 방식의 툴 구현체
  ├─ registry.py          모든 툴의 중앙 등록 테이블 (스키마 + 핸들러)
  ├─ approval.py          위험 명령어 탐지 및 세션별 승인 게이트
  ├─ delegate_tool.py     서브에이전트 생성/병렬 작업 위임
  └─ environments/        터미널 실행 백엔드 (local, Docker, SSH, Singularity, Modal, Daytona 등)

gateway/            메시징 게이트웨이
  ├─ run.py              GatewayRunner — 플랫폼 생명주기, 메시지 라우팅, cron 배포
  ├─ session.py          세션 저장소, 컨텍스트 프롬프트, 리셋 정책
  └─ platforms/          플랫폼 어댑터 일부 + 공통 베이스(base.py), ADDING_A_PLATFORM.md

plugins/            확장 플러그인 (코어를 건드리지 않고 기능을 붙이는 곳)
  ├─ platforms/          대부분의 메신저 어댑터 (telegram, discord, slack, whatsapp, matrix 등)
  ├─ model-providers/    추론 백엔드 (openrouter, anthropic 등)
  ├─ memory/             메모리 프로바이더 (honcho, mem0 등 — 신규 추가는 별도 저장소로)
  └─ kanban/, context_engine/, image_gen/ ...
providers/          모델 프로바이더 레거시/베이스 (base.py)

ui-tui/             Ink(React) 기반 TUI — `hermes --tui`
tui_gateway/        TUI·대시보드·데스크톱용 Python JSON-RPC 백엔드
web/                웹 대시보드 (`hermes dashboard`, /chat은 실제 TUI를 PTY로 임베드)
apps/               Electron 데스크톱 앱(desktop), 공용 전송 계층(shared), 설치기
acp_adapter/        ACP 서버 — VS Code / Zed / JetBrains 연동 (`hermes-acp`)
cron/               스케줄러 (jobs.py, scheduler.py)

skills/             번들 스킬 (설치 시 ~/.hermes/skills/ 로 복사)
optional-skills/    무겁거나 특수한 공식 옵션 스킬 (`hermes skills install`로 옵트인)
scripts/            설치 스크립트 + 테스트 러너(run_tests.sh)
tests/, tests-js/   Python / JS·TS 테스트 스위트
website/            Docusaurus 문서 사이트
cli-config.yaml.example   설정 예시 (설치 시 ~/.hermes/config.yaml 로 복사)
AGENTS.md           개발 가이드 및 기여 규칙 (필독)
```

> **꼭 알아야 할 설계 원칙** (자세한 내용은 `AGENTS.md`)
> - **프롬프트 캐시는 신성하다**: 대화 도중 시스템 프롬프트, toolset, 과거 메시지를 바꾸지 않습니다. 예외는 컨텍스트 압축뿐입니다.
> - **좁은 코어**: 툴 스키마는 매 API 호출마다 전송됩니다. 새 기능은 기존 코드 확장 → CLI 명령과 스킬 → `check_fn`으로 조건부 노출되는 툴 → 플러그인 → MCP 순으로 먼저 검토하고, 새 코어 툴은 최후 수단입니다(Footprint Ladder).
> - **프로필 안전 경로**: 상태 경로는 항상 `get_hermes_home()`으로 만들고, `~/.hermes`를 하드코딩하지 않습니다.
> - **설정 분리**: `config.yaml`에는 일반 설정을, `.env`에는 API 키 같은 비밀값만 둡니다.

---

## 3. 핵심 파일 5개와 역할

| # | 파일 | 역할 |
|---|------|------|
| 1 | `agent/conversation_loop.py` | `run_conversation()`의 실제 구현체로, "LLM 호출 → tool_calls 있으면 실행 → 없으면 종료" 루프와 재시도·fallback·캐시 처리가 모두 여기 있습니다(`while` 루프는 1415행 부근). |
| 2 | `run_agent.py` | 모든 진입점이 모이는 `AIAgent` 클래스로, 모델·프로바이더·toolset·세션·예산·콜백 같은 상태를 보관하고 초기화와 루프는 `agent/`로 위임합니다. |
| 3 | `model_tools.py` | 활성 toolset에 맞춰 모델에 보낼 툴 스키마를 만들고(`get_tool_definitions`), 모델이 요청한 툴 호출을 실제 핸들러로 연결합니다(`handle_function_call`). |
| 4 | `agent/context_compressor.py` | 컨텍스트 한도에 가까워지면 과거 메시지를 요약해 줄이는 압축 엔진으로, 대화 도중 과거 컨텍스트를 바꿀 수 있는 유일한 경로입니다. |
| 5 | `hermes_state.py` | SQLite + FTS5 기반 `SessionDB`로, 대화 기록·메타데이터·검색·게이트웨이 라우팅을 맡아 세션 간 연속성을 보장합니다. |

**다음으로 볼 파일**
- `tools/registry.py`: 툴 중앙 레지스트리
- `gateway/run.py`: 메신저 게이트웨이 오케스트레이터
- `agent/prompt_builder.py`: 시스템 프롬프트 조립
- `hermes_cli/main.py`: `hermes` 명령 진입점

---

## 4. 주요 코드 흐름

### 4-0. 진입점

`pyproject.toml`의 `[project.scripts]`에 등록된 명령은 세 가지입니다.

| 명령 | 진입 함수 | 용도 |
|---|---|---|
| `hermes` | `hermes_cli.main:main` | 메인 CLI (chat, gateway, setup, cron 등 모든 서브커맨드) |
| `hermes-agent` | `run_agent:main` | 에이전트 단독 실행 (질의 1건) |
| `hermes-acp` | `acp_adapter.entry:main` | IDE용 ACP 서버 |

`hermes`를 실행하면 `main()`보다 **import 시점에 실행되는 코드**가 먼저 돕니다.

```
hermes_cli/__init__.py   _ensure_utf8()  — stdio를 UTF-8로
hermes_cli/main.py (모듈 최상단)
  ├─ hermes_bootstrap, 깨진 venv 자동 복구
  ├─ _apply_profile_override()  — -p/--profile을 먼저 읽어 HERMES_HOME 설정 (가장 중요)
  └─ load_hermes_dotenv()       — ~/.hermes/.env 로드
main()
  └─ argparse → args.func(args)
       └─ (서브커맨드 없음) cmd_chat → cli.main() 또는 TUI
```

### 4-1. 대화 실행 흐름 (공통 경로)

```
사용자 입력 (CLI / 메신저 / TUI / 데스크톱 / REST API / ACP)
        │
        ▼
AIAgent.run_conversation()   ← run_agent.py:7759 (forwarder)
        ▼
agent.conversation_loop.run_conversation()
        │
        ├─ build_turn_context()        (agent/turn_context.py)
        │     시스템 프롬프트 생성/복원(세션당 1회), 사용자 메시지 추가,
        │     DB 세션 행 생성, preflight 압축, pre_llm_call 훅, 메모리 prefetch
        │
        ├─ while (반복 한도·예산 이내):
        │     ├─ 인터럽트 확인
        │     ├─ API 메시지 조립 (역할 교대 복구, 메모리/플러그인 컨텍스트는 user 쪽에 주입,
        │     │                    고아 tool 결과 정리, 캐시 마커)
        │     ├─ 압축 필요 여부 검사 → 필요하면 압축
        │     ├─ LLM 호출 (스트리밍/비스트리밍, 재시도·credential 갱신·fallback 프로바이더)
        │     ├─ tool_calls 있음 → agent/tool_executor.py (안전한 툴은 최대 8개 병렬)
        │     │        → model_tools.handle_function_call()
        │     │        → tools/registry.py 디스패치 → 실제 툴 실행
        │     │        → 결과를 tool 메시지로 추가, 반복 계속
        │     └─ tool_calls 없음 → 최종 응답으로 루프 종료
        │
        └─ finalize_turn()             (agent/turn_finalizer.py)
              trajectory 저장, VM/브라우저 정리, 세션 DB 저장,
              transform_llm_output / post_llm_call 훅, 외부 메모리 sync
        ▼
결과 dict 반환 (final_response, messages, completed, api_calls)
```

이 루프에 도달하는 진입점은 여러 개입니다(대화형 CLI, one-shot CLI, 메시징 게이트웨이, TUI, REST API, 서브에이전트 위임, 배치 실행, 큐레이터/백그라운드 리뷰, ACP). 하지만 **실제 대화 루프 구현은 `agent/conversation_loop.py` 하나뿐**입니다. 루프 코드의 대부분은 오류 처리, fallback, 압축이 차지합니다.

### 4-2. `python run_agent.py` 직접 실행 시 흐름

1. **import 시점**
   - `hermes_bootstrap`: Windows UTF-8 설정
   - `.env` 로드: `~/.hermes/.env`, 없으면 저장소의 `.env`
   - `from model_tools import ...`: `discover_builtin_tools()`가 `tools/*.py`를 import해 툴을 등록하고, `discover_plugins()`가 플러그인을 불러옵니다.
2. `fire.Fire(main)`이 CLI 인자를 `main()`의 함수 인자로 매핑합니다. `--list_tools`를 주면 목록만 출력하고 종료합니다.
3. `main()`이 `AIAgent(...)`를 생성하면 `agent/agent_init.py::init_agent`가 다음 순서로 초기화합니다.
   1. 반복 예산
   2. 프로바이더와 api_mode 결정
   3. LLM 클라이언트 생성과 fallback 체인 구성
   4. **툴 스키마 확정**: 세션 동안 고정됩니다.
   5. 체크포인트, 세션 DB, todo
   6. 메모리
   7. 컨텍스트 엔진/압축기
4. `agent.run_conversation(user_query)`가 4-1의 흐름을 실행합니다.
5. 결과 요약을 출력하고, `--save_sample`이면 trajectory JSON도 저장합니다.

> **주의**
> - `run_agent.py`를 직접 실행하거나 `from run_agent import AIAgent`로 import하면 `_apply_profile_override()`가 실행되지 않습니다. 특정 프로필을 쓰려면 실행 전에 `HERMES_HOME`을 직접 설정하세요.
> - 반복 한도 기본값이 경로마다 다릅니다. `AIAgent`의 기본 `max_iterations`는 90이고, `run_agent.py`의 `main()`은 `--max_turns` 기본값 10을 넘깁니다. (`AGENTS.md`의 예시에는 500으로 적혀 있습니다.)

### 4-3. 툴 추가 방식

먼저 **정말 코어 툴이어야 하는지** 판단하세요. 개인용이나 특정 사용자용 기능이라면 플러그인(`~/.hermes/plugins/<name>/`에서 `ctx.register_tool(...)`)이나 스킬로 만드는 것이 원칙입니다.

코어 툴을 추가할 때는 **두 파일**을 수정합니다.
1. `tools/<name>_tool.py`: 모듈 최상위에서 `registry.register(...)`를 호출합니다. `model_tools.py`가 import할 때 자동으로 수집합니다.
2. `toolsets.py`: `_HERMES_CORE_TOOLS` 또는 특정 toolset에 툴 이름을 추가합니다. **이 단계를 빼면 툴이 등록만 되고 모델에는 노출되지 않습니다.**

핸들러는 반드시 JSON 문자열을 반환해야 합니다. 프롬프트 캐시를 지키기 위해 대화 도중에는 toolset을 바꾸지 않습니다.

---

## 5. 개발 시작하기

### 5-1. 사전 요구사항

| 항목 | 비고 |
|---|---|
| Git | `git-lfs` 확장 포함 |
| Python 3.11–3.13 | 없으면 uv가 자동 설치 |
| uv | 빠른 Python 패키지 매니저 ([설치 가이드](https://docs.astral.sh/uv/)) |
| Node.js 20+ | 선택 — 브라우저 툴, WhatsApp 브릿지, TUI/데스크톱 개발에 필요 |

### 5-2. 설치 (표준 설치 스크립트 방식, 권장)

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
cd "${HERMES_HOME:-$HOME/.hermes}/hermes-agent"

# 개발/테스트용 extras 추가 설치
uv pip install -e ".[all,dev]"

# 선택: 브라우저 툴 / 문서 사이트 의존성
npm install
```

> ⚠️ **주의**: 소스 트리 안에 venv를 만들지 마세요. 에이전트가 자기 자신의 체크아웃에 대해 `rm -rf venv` 같은 상대경로 명령을 실행하면 실행 중인 런타임이 삭제될 수 있습니다. 수동 클론 방식을 쓸 경우 venv는 반드시 트리 **바깥**(`~/.hermes/venvs/hermes-dev` 등)에 생성하세요.

### 5-3. 개발 환경 설정

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env

# 최소한 LLM provider 키 하나는 필요 (.env에는 비밀값만!)
echo "OPENROUTER_API_KEY=***" >> ~/.hermes/.env
```

타임아웃, 기능 플래그 같은 일반 설정은 `.env`가 아니라 `~/.hermes/config.yaml`에 넣습니다.

### 5-4. 실행

```bash
hermes doctor                 # 환경 진단
hermes                        # 대화형 CLI
hermes --tui                  # Ink TUI
hermes chat -q "Hello"        # 간단한 one-shot 대화
hermes logs --follow          # 로그 확인 (~/.hermes/logs/)

# 에이전트 단독 실행 (run_agent.py)
python run_agent.py --list_tools                       # 툴/toolset 목록 (API 키 불필요)
python run_agent.py --query "..." --max_turns 5        # 질의 1건 실행
python run_agent.py --query "..." --enabled_toolsets=web,file --save_sample
```

### 5-5. 테스트

Python 테스트는 **반드시 `scripts/run_tests.sh`로 실행**하세요. `pytest`를 직접 실행하지 마세요. 이 스크립트는 CI와 같은 환경을 만듭니다(API 키 환경변수 제거, `TZ=UTC`, `LANG=C.UTF-8`, 임시 `HERMES_HOME`, 테스트 파일마다 별도 서브프로세스). 직접 `pytest`를 돌리면 "로컬에선 통과, CI에선 실패" 같은 문제가 생깁니다.

```bash
scripts/run_tests.sh                                    # 전체 스위트
scripts/run_tests.sh tests/gateway/                     # 디렉토리 단위
scripts/run_tests.sh tests/agent/test_foo.py -k test_x  # 단일 테스트 (파일 + -k)

# TUI (TypeScript)
cd ui-tui && npm test
```

테스트 작성 규칙:
- 현재 값을 그대로 박제하는 change-detector 테스트(모델 목록, 버전 숫자, 개수 비교)는 쓰지 않습니다. 대신 두 데이터가 어떻게 관계해야 하는지(불변식)를 검증합니다.
- 소스 파일 텍스트를 읽어 검사하는 테스트는 금지입니다.
- 테스트에서 `~/.hermes/`에 쓰지 않습니다.

### 5-6. 브랜치/PR 흐름

```bash
git checkout -b fix/description
# ... 작업 ...
scripts/run_tests.sh
```

자세한 기여 규칙, 스타일 가이드, 아키텍처 상세는 `AGENTS.md`와 `CONTRIBUTING.md`를 참고하세요.
