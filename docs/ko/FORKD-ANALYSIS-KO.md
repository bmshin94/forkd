# forkd 전수조사 분석 리포트 (한국어)

> 이 문서는 `bmshin94/forkd` 저장소를 전수조사하여 **forkd가 무엇인지, 언제
> 쓰는지, 어떤 도움이 되는지, 어떻게 수익화할 수 있는지**를 한국어로 정리한
> 리포트입니다. 대화 세션에서 다룬 질문/답변을 문서 형태로 재구성했습니다.

**작성일**: 2026-09-27
**분석 대상 버전**: v0.5.3 (Alpha)
**분석 브랜치**: `claude/determined-euler-z205gl`

---

## 📍 GitHub 주소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/forkd |
| **원본(업스트림) 저장소** | https://github.com/deeplethe/forkd |
| 벤더링된 Firecracker 포크 | https://github.com/deeplethe/firecracker/tree/forkd-v0.4-mem-backend-shared-v1.12 |
| Python SDK (PyPI) | https://pypi.org/project/forkd/ |
| MCP 서버 (PyPI) | https://pypi.org/project/forkd-mcp/ |
| TypeScript SDK (npm) | `@deeplethe/forkd` |
| 릴리스 바이너리 | https://github.com/deeplethe/forkd/releases |
| 스냅샷 허브 인덱스 | `raw.githubusercontent.com/deeplethe/forkd/main/registry.json` |

### 비교 대상으로 언급된 프로젝트

- CubeSandbox — https://github.com/TencentCloud/CubeSandbox
- Daytona — https://github.com/daytonaio/daytona
- OpenSandbox — https://github.com/alibaba/OpenSandbox
- E2B — https://github.com/e2b-dev/E2B · https://github.com/e2b-dev/infra
- BoxLite — https://github.com/boxlite-ai/boxlite
- Firecracker (업스트림) — https://github.com/firecracker-microvm/firecracker

---

## 1. 이게 뭐 하는 건가 (요약)

| 항목 | 내용 |
|---|---|
| 이름 | `forkd` |
| 한 줄 정의 | **AI 에이전트 팬아웃(fan-out)용 microVM 샌드박스 런타임** |
| 기반 기술 | Firecracker + KVM + Linux Copy-on-Write(`mmap MAP_PRIVATE`) |
| 언어 | Rust 약 25,400줄 + Python/TypeScript SDK 약 1,900줄 |
| 라이선스 | Apache 2.0 (상업적 이용 가능, 소스 공개 의무 없음) |
| 상태 | Alpha — 디스크 포맷/API 변경 가능 |
| 헤드라인 | **"Fork 100 microVMs in 101 ms. BRANCH a live VM in 56 ms"** |

### 핵심 아이디어

```
1단계  부모 VM을 딱 한 번 부팅해 예열(python 실행, numpy import, 모델 로딩)
       → 일시정지 후 디스크에 스냅샷 저장 (memory.bin + vmstate)

2단계  자식 VM들은 그 memory.bin을 mmap(MAP_PRIVATE)으로 매핑
       → 커널이 페이지 단위 Copy-on-Write 수행
       → 읽기만 하면 메모리 추가 0, 쓰는 페이지만 개별 복사
       → 결과: 자식 1개당 추가 메모리 0.12 MiB
```

비유: 피자 도우를 한 판 미리 구워두고, 손님 100명에게 그 도우를 "복사"해서
토핑만 다르게 얹는 것. 매번 반죽부터 시작하지 않는다.

### 성능 비교 (README 실측 / N=100)

| 런타임 | 100개 띄우기 | 샌드박스당 메모리 |
|---|---:|---:|
| **forkd** | **101 ms** | **0.12 MiB** |
| CubeSandbox (fast path) | 1.06 s | 5 MiB |
| Firecracker 생 콜드부팅 | 759 ms | 84 MiB |
| BoxLite | 113.2 s | — |
| OpenSandbox (Docker 런타임) | 122.0 s | — |
| gVisor (runsc) | 288.6 s | — |
| Docker (runc) | 335.3 s | 4 MiB |

> 주의: 이 표는 **forkd의 fork-from-warm**과 **다른 프로젝트의 cold-start**를
> 비교한 것이다. README 각주도 "different operating points by design"이라고
> 인정하고 있다. 자기 하드웨어에서는 `forkd bench`로 직접 측정해야 한다.

### 같은 계산, 두 가지 경로 (예열 상속의 가치)

| 호출 | 시간 | 설명 |
|---|---:|---|
| `sandbox.eval("numpy.zeros(5).tolist()")` | 1 ms | 예열된 PID 1의 파이썬 재사용 |
| `sandbox.commands.run("python3 -c '...'")` | 96 ms | 새 프로세스가 numpy를 다시 import |

96배 차이. 이 표가 프로젝트의 존재 이유를 한 줄로 설명한다.

---

## 2. 킬러 기능: BRANCH (실행 중 VM 분기)

일반 스냅샷은 "초기 예열 상태"에서만 복제할 수 있지만, forkd의 `BRANCH`는
**실행 중인 VM을 도중에 멈추고 분기**한다. 에이전트가 "생각하는 중간에"
갈라질 수 있다.

```python
from forkd import Controller
c = Controller()

# live_fork=True 필수 (memfd 기반 RAM — UFFD_WP의 전제조건)
parent = c.spawn_sandboxes("pyagent", n=1, live_fork=True)[0]

# ... 에이전트가 추론하고, 파일 만들고, DB 채우고 ...

# 위험한 단계 직전에 체크포인트 분기
branch = c.branch_sandbox(parent["id"], mode="live", wait=False)

# 그 시점 상태를 온전히 물려받은 자식 5개 → 5가지 전략 병렬 시도
kids = c.spawn_sandboxes(branch["tag"], n=5)
```

### 게임 세이브파일 비유

```
        에이전트가 작업 중 (파일 40개, DB, 50MB 바이너리, 추론 15단계)
                    │  ← 여기서 BRANCH (세이브) 56 ms
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    자식A        자식B        자식C
  "리팩터링"   "패치만"    "테스트추가"

  · 3명 모두 그 40개 파일 + DB + 50MB 바이너리 + 추론 15단계를 이미 보유
  · 서로의 수정사항은 완전 격리 (__pycache__ 포함)
  · 셋 다 실패하면 세이브 지점에서 다시 분기 (LLM 재호출 비용 0)
```

### 프롬프트로 대체 불가능한 이유

| 프롬프트로 복제 가능 | forkd만 복제 가능 |
|---|---|
| 대화 내용, 추론 텍스트, 지시사항 | 파일시스템 전체 상태 |
| | 50 MiB 바이너리 파일 |
| | 이미 import된 메모리 |
| | 실행 중 프로세스 (Chromium, Postgres) |
| | `__pycache__`, DB 인덱스, JIT 컴파일 결과 |

> 저장소의 표현: **"Bytes can't fit in a prompt"**

### 실제 데모 (저장소 내장)

- `recipes/langgraph-react/` — 교토 여행 일정 ReAct 에이전트. 부모가 추론 중간에
  BRANCH되고, 손자 3명이 각기 다른 힌트를 받음. 부모는 "니시키 시장"을 1일차로
  골랐는데 **힌트 받은 자식 3명 전원이 독립적으로 "아라시야마 대나무숲"으로
  교체**. 장소 교체를 지시한 적은 없다.
- `recipes/coding-agent-fork/` — "그냥 LLM 3번 병렬 호출하면 되지 않나?"에 대한
  반박. 50 MiB 바이너리가 BRANCH 한 번으로 4개 샌드박스에 바이트 단위 동일하게
  전달되고, 손자 3명의 수정사항은 완전 격리됨.

### BRANCH 속도 진화사

| 버전 | 방식 | 부모 멈춤 시간 | 개선 |
|---|---|---:|---|
| v0.2 | Full 스냅샷 | 29.3 s | — (사실상 사용 불가) |
| v0.3 | Diff 스냅샷 | 205 ms | **143×** |
| v0.3.1 | 멀티 BRANCH 체인 | — | 5연속 시 14× 누적 |
| v0.3.4 | `posix_fallocate` 30줄 | ~150 ms (6연속 유지) | **17.6×** |
| v0.4 | memfd + UFFD_WP (live) | **56 ms p50 / 64 ms p90** | **3.6×** |
| v0.4 `wait:false` | 비동기 백그라운드 복사 | 호출자 ~10 ms 반환 | 체감 **200×** |

- v0.3.4 버그의 원인은 **ext4 writeback throttle 누적** — 같은 부모를 6번
  연속 BRANCH하면 150 ms → 2.7 s로 부풀어올랐다. 30줄 수정으로 해결.
- v0.4는 `mem_backend.backend_type: "Shared"`가 업스트림 Firecracker에 없어서
  **벤더링된 포크**가 필요하다. 업스트림 제안서는
  `FIRECRACKER-UPSTREAM-PROPOSAL.md`에 있다.

### v0.5: diff 스냅샷 체인

`pip install numpy / pandas / sklearn`을 1.5 GiB 복사 3개가 아니라 Docker
레이어처럼 쌓는다. 512 MiB 베이스, ext4, i7-12700 기준:

| 체인 헤드 | 깊이 | spawn p50 | 링크당 세금 |
|---|---:|---:|---:|
| base (flat) | 0 | 59 ms | — |
| `+numpy` | 1 | 751 ms | +692 ms |
| `+pandas` | 2 | 1,222 ms | +471 ms |
| `+sklearn` | 3 | 1,668 ms | +446 ms |

링크당 세금은 베이스의 SHA-256 검증 비용(512 MiB @ 1.1 GiB/s ≈ 460 ms)을
따라간다. mmap-once-then-incremental-verify 최적화는 v0.6 예정.
정확성은 L1/L2/L3 + flat-equivalent에서 **90/90 (100%) 통과**.

---

## 3. 폴더별 구조 분석

```
forkd/
├── crates/                    Rust 본체
│   ├── forkd-vmm/       (5,661줄) Firecracker 래퍼: BootConfig, Vm, Snapshot, cgroup
│   │   ├── lib.rs       (4,572줄) VM 생명주기 전체
│   │   ├── chain.rs     (588줄)   v0.5 diff 스냅샷 체인
│   │   └── memfd.rs     (501줄)   v0.4 live fork용 memfd
│   ├── forkd-controller/(7,000줄+) 데몬: REST, 레지스트리, 인증, 감사로그
│   │   ├── http.rs      (5,028줄) REST 엔드포인트
│   │   ├── state.rs     (2,158줄) 스냅샷/샌드박스 레지스트리
│   │   ├── netns.rs     (418줄)   자식별 네트워크 namespace
│   │   ├── audit.rs     (265줄)   append-only JSON 감사 로그
│   │   └── auth.rs      (204줄)   Bearer 토큰 (상수시간 비교)
│   ├── forkd-cli/       (8,600줄) `forkd` 바이너리, 30개 서브커맨드
│   │   ├── main.rs      (5,366줄)
│   │   ├── hub.rs       (1,732줄) pack/unpack/pull/push
│   │   └── doctor.rs    (904줄)   호스트 17개 항목 진단
│   └── forkd-uffd/      (1,076줄) userfaultfd write-protect 스냅샷
├── sdk/
│   ├── python/          E2B 호환 SDK (`from forkd import Sandbox`)
│   ├── typescript/      @deeplethe/forkd
│   └── mcp/             forkd-mcp — MCP 서버 (353줄, 툴 12개)
├── recipes/             18개 레시피 (프레임워크 연동 + rootfs 빌드 + 특수용도)
├── bench/               벤치 하네스 + 경쟁사 실측 방법론 문서
├── docs/                API.md, SECURITY.md, RUNBOOK.md, HUB.md, 설계문서 3편
├── scripts/             호스트 셋업 (KVM, 커널, tap, netns, rootfs)
├── packaging/           systemd 유닛 + K8s 매니페스트 + Arch PKGBUILD
├── experiments/         v0.4 PoC 4종 (UFFD-WP, THP, KVM, restore)
├── rootfs-init/         게스트 PID 1 스크립트 + :8888 에이전트 서버
├── registry.json        스냅샷 허브 인덱스 (사전제작 9종)
└── DESIGN*.md (6개)     설계문서 약 98 KB
```

### 레시피 목록

**프레임워크 연동 (호스트 측 파이썬, 각 150~250줄, `--dry-run` 지원)**

| 레시피 | 드라이버 | forkd 고유 포인트 |
|---|---|---|
| `mcp-agent/` | Claude Desktop / Cursor / Cline | MCP 프로토콜 E2E 검증 |
| `crewai-fanout/` | CrewAI | 에이전트 N개 = microVM N개, 자식당 ~24 ms |
| `autogen-branch/` | AutoGen | forkd 기반 `CodeExecutor` + 대화 중 BRANCH |
| `openai-swarm/` | OpenAI Swarm / Agents SDK | 핸드오프 = BRANCH |
| `langgraph-react/` | LangGraph | 프론트페이지 데모, 추론 중 분기 |
| `speculative-agent/` | — | 투기적 실행 패턴 |

**rootfs / 특수용도**

| 레시피 | 언제 쓰나 |
|---|---|
| `python-numpy/` | 벤치마크 재현, 가장 가벼운 Python + numpy |
| `e2b-codeinterpreter/` | AI 코드 인터프리터 (E2B SDK 호환) |
| `jupyter-kernel/` | 노트북 / SciPy 스택 사전 import, 커널당 ~1 ms |
| `coding-agent/` | SWE-bench 스타일 코딩 에이전트 (git + 개발도구) |
| `nodejs/` | JS/TS 워크로드, Playwright 팬아웃 |
| `playwright-browser/` | 브라우저 에이전트. 예열 Chromium을 ~10 ms에 fork |
| `agent-workbench/` | 브라우저 + VSCode + Jupyter + MCP 종합 |
| `postgres-fixture/` | 테스트별 격리 Postgres, ~10 ms (vs `initdb` ~2 s) |
| `ci-parallel-pytest/` | pytest 워커 팬아웃, 워커당 ~50 ms |

---

## 4. 설치 및 사용법

### 필수 조건

| 조건 | 이유 |
|---|---|
| x86_64 리눅스 (Ubuntu 22.04+) | Firecracker 요구사항 |
| `/dev/kvm` 사용 가능 | 하드웨어 가상화. 클라우드면 nested virt 필요 |
| cgroup v2 | 자식별 메모리 제한 |
| Linux ≥ 5.7 | v0.4 live BRANCH (`UFFDIO_WRITEPROTECT`) |
| `vm.unprivileged_userfaultfd=1` 또는 `CAP_SYS_PTRACE` | UFFD 접근 |
| root 권한 | KVM / netns / cgroup 조작 |

```bash
# 30초 사전 점검
uname -m                          # x86_64
ls -l /dev/kvm                    # 존재해야 함
grep -cE 'vmx|svm' /proc/cpuinfo  # > 0
stat -fc %T /sys/fs/cgroup        # cgroup2fs
```

> macOS / Windows에서는 직접 동작하지 않는다. 리눅스 VM(중첩 가상화 활성) 또는
> 베어메탈 클라우드(Hetzner, AWS `*.metal`, GCP nested-virt)가 필요하다.
> 대안: 개발은 로컬에서, forkd 데몬은 원격 리눅스 박스에 두고 `FORKD_URL`로
> REST 호출.

### 방법 A — 원샷 (권장)

```bash
curl -sSL https://github.com/deeplethe/forkd/releases/download/v0.5.3/forkd-v0.5.3-x86_64-linux.tar.gz \
  | sudo tar -xz -C /usr/local/bin/

sudo -E forkd quickstart
```

`quickstart`는 호스트 사전점검 → 동의 기반 자동 복구(게스트 커널 다운로드, tap
장치, per-child netns) → `python:3.12-slim`에서 스냅샷 베이크(Docker 없으면
허브에서 pull) → 자식 10개 fork + 타이밍 출력까지 수행한다. 멱등하다.

### 방법 B — 단계별

```bash
sudo bash scripts/setup-host.sh        # KVM + tap (1회). --paranoid 로 체크섬 검증
sudo bash scripts/netns-setup.sh 3     # 자식용 netns
forkd doctor                           # 17개 항목 진단 + 항목별 해결 힌트
pip install forkd                      # Python SDK
npm install @deeplethe/forkd           # TypeScript SDK
pip install forkd-mcp                  # MCP 서버
```

### 방법 C — 허브에서 바로 (~15초)

```bash
forkd pull deeplethe/langgraph-react   # 14.5 MiB, sha256 검증
sudo -E forkd fork --tag langgraph -n 3 --per-child-netns
```

허브 사전제작 스냅샷 9종: `python-numpy`, `playwright-browser`,
`postgres-fixture`, `coding-agent`, `nodejs`, `e2b-codeinterpreter`,
`jupyter-kernel`, `langgraph-react`, `coding-agent-fork`

### 방법 D — 소스 빌드

```bash
sudo bash scripts/setup-host.sh && sudo bash scripts/host-tap.sh
cargo build --release
sudo install -m 0755 target/release/{forkd,forkd-controller} /usr/local/bin/
```

### 데몬 모드 (실서비스)

```bash
sudo install -m 0644 packaging/systemd/forkd-controller.service /etc/systemd/system/
sudo mkdir -p /etc/forkd
sudo bash -c 'head -c 32 /dev/urandom | base64 > /etc/forkd/token'
sudo chmod 600 /etc/forkd/token
sudo systemctl enable --now forkd-controller
```

### 주요 명령어

```bash
# 스냅샷
forkd from-image python:3.12-slim --tag py --extra python3-numpy
forkd snapshot --tag pyagent --kernel ./vmlinux-6.1.141 --rootfs ./py.ext4
forkd snapshot-diff --from py --tag py-numpy --exec "pip install numpy==2.0.2"

# 포크 & 실행
forkd fork --tag py -n 100 --per-child-netns --memory-limit-mib 256
forkd exec --child forkd-child-42 -- "ls /tmp"
forkd eval --child forkd-child-42 -- "numpy.zeros(100).sum()"

# 체인 관리
forkd images
forkd snapshot-info py-numpy                        # 깊이 / 부모 / 의존성
forkd snapshot-compact --from py-pandas --to flat    # 평탄화
forkd rmi py-numpy [--cascade|--force]               # 고아 생성 시 HTTP 409 거부

# 허브
forkd pack --tag py --out py.tar.zst                 # 23배 압축
forkd push --tag py "<presigned-PUT-URL>"
forkd pull https://hub.example.com/py.tar.zst

# 운영
forkd doctor / forkd bench --tag py --n 5 / forkd ls / kill / cleanup
```

### REST API

```bash
TOKEN=$(sudo cat /etc/forkd/token)
curl -H "Authorization: Bearer $TOKEN" -X POST http://127.0.0.1:8889/v1/sandboxes \
  -H 'Content-Type: application/json' \
  -d '{"snapshot_tag":"pyagent","n":5,"per_child_netns":true,"memory_limit_mib":256}'
```

전체 엔드포인트 (`docs/API.md`):

```
GET    /healthz  /version  /metrics
POST   /v1/snapshots            GET/DELETE /v1/snapshots[/:tag]
GET    /v1/snapshots/:tag/info  POST /v1/snapshots/:tag/compact
POST   /v1/sandboxes            GET/DELETE /v1/sandboxes[/:id]
POST   /v1/sandboxes/:id/ping  /exec  /eval  /branch
```

---

## 5. 플러그인인가, 스킬인가, MCP인가?

**셋 다 아니다. forkd는 시스템 데몬 + CLI(인프라 런타임)이고, MCP 서버를
부속으로 함께 제공한다.**

| 형태 | forkd는? | 근거 |
|---|---|---|
| 플러그인 (Claude Code plugin) | ✗ | `.claude-plugin/` 마켓플레이스 구조 없음 |
| 스킬 (Skill) | ✗ | `SKILL.md` 없음. 지시문 묶음이 아니라 실행 인프라 |
| MCP 서버 | 일부만 | `sdk/mcp/`의 `forkd-mcp`가 별도 패키지 |
| 시스템 데몬 + CLI | ✓ | `forkd`, `forkd-controller` 바이너리 |

```
forkd 본체 (Rust 데몬 + CLI, AI와 무관하게 동작)
   ├── REST API      (언어 무관)
   ├── Python SDK    pip install forkd
   ├── TS SDK        npm i @deeplethe/forkd
   └── MCP 서버      pip install forkd-mcp   ← 이 부분만 MCP
```

`forkd-mcp` (`sdk/mcp/forkd_mcp/server.py`, 353줄, FastMCP 기반, stateless):

```
list_snapshots   spawn_sandboxes   branch_sandbox   create_snapshot
wait_for_text    list_sandboxes    get_sandbox      kill_sandbox
exec_command     eval_code         ping_sandbox
```

```jsonc
// claude_desktop_config.json
{ "mcpServers": { "forkd": { "command": "forkd-mcp" } } }
```

환경변수: `FORKD_URL`(기본 `http://127.0.0.1:8889`), `FORKD_TOKEN`,
`FORKD_HTTP_TIMEOUT`. **MCP 서버는 HTTP 클라이언트 래퍼일 뿐이므로 forkd 데몬은
별도로 리눅스 호스트에 떠 있어야 한다.**

---

## 6. API 토큰이 필요한가?

"토큰"이 세 종류로 완전히 다른 얘기다.

### ① forkd 데몬 Bearer 토큰 — 자체 발급, 외부 서비스 무관

```bash
sudo bash -c 'head -c 32 /dev/urandom | base64 > /etc/forkd/token'
sudo chmod 600 /etc/forkd/token
```

`crates/forkd-controller/src/auth.rs` 동작:

- `--token-file` 지정 시 `/healthz` 제외 **모든 요청**에 `Authorization: Bearer <tok>` 필요
- 토큰 파일 없으면 **인증 없이 동작** (localhost 개발용)
- **상수시간 바이트 비교** 구현 → 타이밍 공격 방어
- 토큰은 시작 시 1회만 읽음 → 로테이션은 데몬 재시작 필요
- 비-루프백 배포에는 `--token-file` 필수 (문서 명시)
- TLS는 `rustls` + `axum-server` + `rcgen`

> 과거 보안 이력: `packaging/k8s/`에서 플레이스홀더 토큰이 그대로 통과되던
> HIGH 등급 이슈가 0.1.0~0.1.3에 존재했고 0.1.4에서 수정됨. K8s 매니페스트
> 사용 시 토큰을 반드시 교체해야 한다.

### ② LLM API 키 — 선택사항, 데모 레시피에서만

`OPENAI_API_KEY` / `ANTHROPIC_API_KEY`는 `crewai-fanout`, `autogen-branch`,
`openai-swarm` 등 데모에서만 쓰인다. 모든 프레임워크 레시피에 `--dry-run`
모드가 있어 **LLM 키 없이 forkd 배관만 검증** 가능하다.

### ③ 배포용 토큰 — 배포자일 때만

- `forkd push`: S3/R2 presigned PUT URL
- `forkd pull`: **토큰 불필요**, GitHub Releases 공개 다운로드 + sha256 검증
- PyPI/npm 배포: **Trusted Publishers (OIDC)** — 저장소 시크릿에 토큰 미저장

| 하고 싶은 일 | 필요한 토큰 |
|---|---|
| 로컬에서 써보기 | 없음 |
| 데몬을 원격/공용망 노출 | ① 자체 Bearer 토큰 (필수) |
| 허브에서 받기 | 없음 |
| 허브에 올리기 | presigned URL |
| LLM 에이전트 데모 | ② LLM 키 (또는 `--dry-run`) |

---

## 7. 왜 GitHub에서 주목받을 만한가

> 실제 스타 수는 이 분석에서 확인하지 않았다(원본 저장소가 접근 범위 밖).
> 아래는 **저장소 내용에 근거한 매력 요인 분석**이다.

1. **타이밍 + 시장 공백.** 오픈소스 중 `fork-from-warm`을 실제로 제공하는 유일한
   프로젝트. CubeSandbox는 "coming soon", Daytona/OpenSandbox/E2B/BoxLite는 미지원,
   Modal은 보유하지만 비공개. "Modal이 비공개로 갖고 있던 프리미티브를 Apache 2.0으로"가
   가장 강한 후킹.
2. **헤드라인이 되는 숫자.** "Fork 100 microVMs in 101 ms", "Docker 대비 3,300배",
   "자식당 0.12 MiB", "29.3 s → 205 ms(143×)". GIF 데모 + asciicast 원본 +
   재현 스크립트(`bench/bench-spawn-100.sh`)까지 제공.
3. **정직함.** `Where forkd is wrong` / `Where the others fit better` 섹션을
   README에 직접 둔다. CubeSandbox를 20.3 s로 잘못 측정했던 것을 "우리 설정
   미스였다"고 정정하고 1.06 s로 갱신(`bench/CUBESANDBOX.md`). 경쟁사 버그를
   찾아 업스트림 PR(#236/#237)까지 제출. Phase 6 측정이 게스트 Oops로 오염됐다는
   것도 실토. Alpha·멀티노드 부재·보안감사 미완을 명시.
4. **엔지니어링 서사.** 29.3 s → 205 ms → (6연속 BRANCH가 2.7 s로 부푸는 버그)
   → ext4 writeback throttle 원인 규명 → `posix_fallocate` 30줄로 17.6× →
   그래도 부족해서 memfd+UFFD_WP로 56 ms → Firecracker가 MAP_SHARED를 안 줘서
   포크 → 업스트림 제안서 작성. "30줄 커밋으로 17.6배"는 강한 이야기다.
5. **진입장벽을 계속 낮춤.** ROADMAP M1 목표가 "Widen the entry point".
   `forkd quickstart` 한 줄, `forkd doctor` 17개 진단 + 항목별 힌트,
   `forkd pull`로 10분 rootfs 빌드를 15초로, 레시피 18개, `--dry-run`.
6. **문서 품질.** README 42 KB(890줄), 중국어 README 35 KB, CHANGELOG 45 KB,
   DESIGN 문서 6편 약 98 KB. "Enterprise deployment FAQ"에 K8s 배포, Pod당
   사이징(vCPU당 활성 1개 / 8 GiB당 유휴 50개), 게스트/호스트 커널 매트릭스까지.
7. **CI 보안 디테일.** `pr-policy.yml`에서 `pull_request_target` 사용 이유를
   "정책은 보호된 base 브랜치에서 와야 하고 PR 내용에서 오면 안 된다"고 주석으로
   설명.

**균형 잡힌 시각**: 프로덕션 도입 사례는 검증되지 않았고, x86_64 리눅스 + KVM
전용이라 실사용자 풀이 좁다. 벤치마크는 자신에게 유리한 비교 축을 쓴다(각주로
인정). 스타 수가 곧 프로덕션 채택을 뜻하지도 않는다.

---

## 8. 로컬 에이전트 구축에 도움이 되는가

**된다. 단, 하드웨어 조건이 붙는다.**

| 환경 | 판정 |
|---|---|
| 리눅스 데스크탑/서버 (x86_64) + KVM | 가능 |
| Hetzner / OVH 베어메탈 | 가능 (가성비 좋음) |
| AWS `*.metal`, GCP nested-virt | 가능하지만 비용 높음 |
| WSL2 | 중첩 가상화 설정 필요 |
| macOS (Apple Silicon) | 직접 불가 |
| 일반 VPS (nested virt 없음) | 불가 |

### 도움되는 지점

1. **격리된 코드 실행 백엔드** — LLM이 생성한 코드를 KVM 하드웨어 격리 안에서
   실행. VM을 나가면 호스트에 흔적이 남지 않는다.
2. **예열 상속** — `import torch` 3~8초가 `eval()` 1 ms로. 도구를 30번 호출하는
   에이전트라면 90~240초 → 0.03초.
3. **투기적 실행** — BRANCH 후 N가지 전략을 동시에 시도하고 최선을 선택.
   `recipes/speculative-agent/`, `recipes/openai-swarm/`("핸드오프 = BRANCH").
4. **롤백 가능한 에이전트** — 마이그레이션 전 BRANCH, 망가지면 복귀.
   LLM 재호출 비용 0.
5. **프레임워크 연동 완비** — LangGraph / CrewAI / AutoGen / OpenAI Swarm / MCP.

### 도움 안 되는 경우

- API만 호출하고 코드 실행이 없는 에이전트 (오버엔지니어링)
- 데스크탑 앱 자동화 에이전트 (forkd는 리눅스 VM)
- 동시 실행이 5개 미만 (Docker로 충분)
- 팀에 리눅스/KVM 운영 역량이 없을 때 (`sudo`, netns, cgroup, 커널 버전 관리)

### 현실적 로드맵

```
1주차  리눅스 박스 확보 → forkd quickstart → doctor → bench (내 하드웨어 실측)
2주차  recipes/mcp-agent/ 로 MCP 연결 → 클로드가 직접 microVM에 코드 실행
3주차  recipes/coding-agent/ 기반으로 자체 에이전트 (E2B 사용자는 import 한 줄 교체)
4주차  BRANCH 도입 → 투기적 실행 실험 (recipes/speculative-agent/)
```

---

## 9. React나 PHP로 만들 수 있는가

### 코어는 불가능

| forkd가 하는 일 | 필요한 것 | React/PHP |
|---|---|---|
| `/dev/kvm` ioctl로 VM 생성 | 저수준 시스템콜 | 불가 |
| `mmap(MAP_PRIVATE)` CoW 매핑 | 메모리 관리 시스템콜 | 불가 |
| `userfaultfd` + `UFFDIO_WRITEPROTECT` | 커널 페이지폴트 핸들링 | 불가 |
| `memfd_create` 공유 메모리 | 저수준 FD | 불가 |
| cgroup v2 `memory.max` 조작 | `/sys/fs/cgroup` 쓰기 + root | 불가 |
| netns + veth 생성 | `CLONE_NEWNET`, netlink | 불가 |
| Firecracker 프로세스/vsock 관리 | 프로세스·소켓 제어 | 불가 |
| `FICLONE` ioctl (reflink) | 파일시스템 ioctl | 불가 |

언어 선택은 취향이 아니라 물리적 필요다. Rust는 메모리 안전 + GC 없는 예측가능한
레이턴시(101 ms의 전제). Go는 GC pause가 ms 단위 레이턴시를 망친다. React는
브라우저 샌드박스 안이라 커널 접근 개념이 없고, PHP는 요청-응답 모델이라 microVM
100개의 생명주기를 소유할 수 없다.

### 그 위의 레이어는 전부 가능하다

```
React 프론트엔드   실시간 대시보드, BRANCH 트리 시각화, 스냅샷 체인 인스펙터,
                  xterm.js 터미널, Prometheus 차트, Monaco 웹 IDE
      │ HTTP
PHP/Laravel 백엔드 인증, 멀티테넌시/쿼터, 사용량 미터링 + 결제, 팀/권한/감사,
                  멀티노드 스케줄링(현재 공백!), 웹훅/잡큐
      │ REST (Bearer)
forkd-controller   POST /v1/sandboxes  /branch  /exec ...  ← 그대로 사용
```

### PHP 클라이언트 (공식 SDK 없음 — 선점 기회)

```php
<?php
class ForkdClient {
    public function __construct(
        private string $baseUrl = 'http://127.0.0.1:8889',
        private string $token = ''
    ) {}

    private function req(string $method, string $path, ?array $body = null): mixed {
        $ch = curl_init($this->baseUrl . $path);
        $headers = ['Content-Type: application/json'];
        if ($this->token) $headers[] = "Authorization: Bearer {$this->token}";
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST  => $method,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER     => $headers,
            CURLOPT_POSTFIELDS     => $body ? json_encode($body) : null,
            CURLOPT_TIMEOUT        => 60,
        ]);
        $res  = curl_exec($ch);
        $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        if ($code >= 400) throw new RuntimeException("forkd $code: $res");
        return json_decode($res, true);
    }

    public function spawn(string $tag, int $n = 1, array $opts = []): array {
        return $this->req('POST', '/v1/sandboxes',
            array_merge(['snapshot_tag' => $tag, 'n' => $n,
                         'per_child_netns' => true], $opts));
    }
    public function exec(string $id, string $cmd): array {
        return $this->req('POST', "/v1/sandboxes/$id/exec", ['cmd' => $cmd]);
    }
    public function eval(string $id, string $code): array {
        return $this->req('POST', "/v1/sandboxes/$id/eval", ['code' => $code]);
    }
    public function branch(string $id, array $opts = []): array {
        return $this->req('POST', "/v1/sandboxes/$id/branch",
            array_merge(['mode' => 'live', 'wait' => false], $opts));
    }
    public function kill(string $id): void { $this->req('DELETE', "/v1/sandboxes/$id"); }
    public function listSnapshots(): array { return $this->req('GET', '/v1/snapshots'); }
}
```

### React BRANCH 트리 뷰어 (아직 아무도 안 만든 영역)

`forkd snapshot-info`가 `chain depth`, `parent_tag`, `ancestors`, `dependents`를
제공하므로 데이터는 준비돼 있다. React Flow + SWR로 계보 그래프, 노드 클릭 시
fork, 삭제 시 고아가 되는 자식 하이라이트 등을 만들 수 있다.

| 레이어 | 언어 | 직접 만들 수 있나 |
|---|---|---|
| KVM/CoW/UFFD 커널 계층 | Rust 필수 | 불가 — 가져다 쓴다 |
| REST 컨트롤러 데몬 | Rust (기존) | 불가 — 가져다 쓴다 |
| 멀티노드 스케줄러 | 무관 | **공백 — 기회** |
| 인증/과금/테넌시 | PHP/Laravel, Node | **공백 — 기회** |
| 웹 대시보드 / BRANCH 트리 | React | **공백 — 기회** |
| PHP SDK | PHP | **없음 — 선점 가능** |

---

## 10. 수익화 아이디어

### 법적 베이스

```
Apache 2.0 — 상업적 이용/수정/재배포 허용, 특허 그랜트 포함,
             소스 공개 의무 없음 (AGPL과의 결정적 차이)
의무       — LICENSE + NOTICE 포함, 변경사항 표시,
             "forkd"/"Deeplethe" 상표를 제품명으로 사용 금지
```

전략적 이점: Daytona는 AGPL-3.0이라 SaaS 제공 시 소스 공개 압박이 있다.
**"AGPL 때문에 Daytona를 못 쓰는 회사"가 forkd의 실질 시장이다.**
(아래 내용은 법률 조언이 아니며, 사업화 전 법률 검토가 필요하다.)

### Tier 1 — 실현 가능성 높고 수요 확실

**① AGPL 탈출구 샌드박스 SaaS**
- 포지셔닝: "E2B 가격의 1/5, 소스 공개 압박 없음, 셀프호스팅 옵션"
- 구조: forkd 노드(Hetzner AX52 급) + PHP/Laravel 컨트롤 플레인 + React 대시보드 + 자체 브랜드
- 가격: Free(월 100 샌드박스-시간) / Pro $29 / Team $199 / Enterprise 연 $10k+
- 원가 마법: 자식당 0.12 MiB이므로 64 GB 서버 하나에 유휴 샌드박스 수백 개 수용
- 난이도 중 / 잠재력 높음 / 초기 자본 월 10만원 이하

**② CI 테스트 가속 서비스 (가장 현실적)**
- 문구: "pytest 스위트를 10배 빠르게. 코드 변경 없이."
- 근거: `postgres-fixture` 2 s → 10 ms(200×), `ci-parallel-pytest` 3 s → 50 ms/워커,
  `playwright-browser` 2~3 s → 10 ms(100~300×)
- 형태: GitHub Action / 셀프호스티드 러너 이미지 / `pytest-forkd` 플러그인
- 가격: 스타터 $99(월 1,000 CI분) / 팀 $499(월 10,000 CI분).
  GitHub Actions 큰 러너 분당 $0.016~0.08 대비 우위
- 난이도 낮음 / 잠재력 높음 / ROI 증명이 쉬움(비용·시간 절감 계산기)

**③ 브라우저 자동화 팜 (Browserless/Browserbase 대안)**
- 근거: 예열 Chromium을 10 ms에 fork
- 시장: AI 웹 스크래핑, computer-use 에이전트, UI 테스트 생성, 웹 리서치
- API: `POST /v1/browsers` + Playwright/Puppeteer CDP 호환
- 가격: 세션당 $0.002 또는 월 $49부터
- 주의: Chromium 부모는 2 GiB+ 필요, 자식 cgroup 상한 2560 MiB+ (ROADMAP M1.1)

### Tier 2 — 개발자 자산 / 간접 수익

**④ 오픈소스 공백 메우기 (자본 0원)**

| 공백 | 만들 것 | 근거 |
|---|---|---|
| PHP SDK 없음 | `forkd-php` (Packagist) | Python/TS만 존재 |
| Go SDK 없음 | `forkd-go` | Go 생태계 규모 |
| 웹 UI 없음 | React 대시보드 + BRANCH 트리 | `snapshot-info`가 데이터 제공 |
| 멀티노드 스케줄러 없음 | 라우터/스케줄러 | README "Status"에 명시된 갭 |
| egress 정책 없음 | netns별 default-deny iptables 도구 | "users add their own" |
| cpu.max/io.max/pids.max 없음 | 쿼터 확장 PR | "not yet in this release" |
| Grafana 대시보드 없음 | `/metrics` 기반 대시보드 JSON | Prometheus 노출 중 |
| Terraform/Ansible 없음 | IaC 모듈 | K8s/systemd만 존재 |

수익 경로: 오픈소스 기여 → 인지도 → 컨설팅(시간당 $100~200) 또는 채용 기회.

**⑤ 스냅샷 마켓플레이스 ("microVM용 Docker Hub")**
- 근거: `registry.json` + `pack/push/pull`이 이미 있고, 현재는 무료 9종뿐
- 상품: 사전예열 ML 스택(PyTorch+CUDA, HF 모델 로딩 완료), 산업별 이미지,
  라이선스 소프트웨어 포함 이미지(라이선스 협의 필수), 컴플라이언스 하드닝 베이스
- 가격: 이미지당 $19~99 또는 큐레이션 월 $29
- 매력: 23배 압축으로 대역폭 비용 작고, sha256 검증 내장으로 신뢰 확보 쉬움

**⑥ Agent Time Machine (가장 독창적)**
- 콘셉트: 에이전트 실행의 "세이브 파일". 어느 시점으로든 롤백 후 분기
- 기능: 실행 타임라인 시각화, 임의 스텝 롤백, 다른 프롬프트/도구/모델로 분기,
  "3단계에서 다르게 했다면?" A/B, LLM 재호출 비용 없는 재실행
- 타깃: 에이전트 개발자, 프롬프트 엔지니어, AI 연구자
- 가격: 개인 $49/월, 팀 $299/월
- 방어력: LangSmith/Langfuse 류 관측 도구는 "기록"만 하고 "되돌아가 재실행"은
  못 한다. 복제하려면 forkd 급 인프라가 필요 — 여기가 moat

### Tier 3 — 자본/전문성 필요

**⑦ 규제 산업 특화 샌드박스** — 금융/헬스케어(HIPAA)/정부. 하드웨어 격리 +
append-only 감사로그 + 온프렘. 연 $50k~500k. 컴플라이언스 인증 + 서드파티
보안 감사 필요(forkd는 아직 미완).

**⑧ LLM 학습 롤아웃 인프라** — RL/RLHF 롤아웃, SWE-bench 평가.
`recipes/coding-agent/`가 정확히 이 모양. 고객은 AI 랩/모델 학습 스타트업.

**⑨ 교육 / 콘텐츠 (자본 0원, 즉시 시작 가능)** — 유튜브 시리즈, 블로그
("`posix_fallocate` 30줄이 17.6배를 만든 이야기"), 유료 강의($199), 전자책.
**한국어 콘텐츠는 경쟁이 거의 없다.**

### 추천 우선순위

```
1순위  CI 테스트 가속 (②)
       고통이 보편적 + 예산 존재 + ROI 증명 쉬움 + 기술 검증됨
       첫 단계: ci-parallel-pytest + postgres-fixture 로 "CI 15분 → 90초" 실측 확보

2순위  오픈소스 공백 메우기 (④)
       자본 0원, 리스크 0, 신뢰 자산 축적
       첫 단계: forkd-php SDK 를 Packagist 등록 + React BRANCH 트리 뷰어

3순위  Agent Time Machine (⑥)
       가장 독창적, 방어 가능한 moat
       첫 단계: recipes/speculative-agent 로 프로토타입
```

### 수익화 전 체크리스트

| 항목 | 상태 | 할 일 |
|---|---|---|
| 라이선스 | OK (Apache 2.0) | LICENSE + NOTICE 동봉, 변경사항 표시 |
| 상표 | 주의 | "forkd" 제품명 금지, 자체 브랜드 필요 |
| 보안 감사 | 미완 | 유료 서비스면 자체 감사 필요 (0.1.x 취약점 이력 참고) |
| Alpha 단계 | 주의 | API/디스크포맷 변경 가능 → 버전 핀 고정 |
| 멀티노드 | 없음 | 확장 시 자체 스케줄러 구현 |
| Egress 정책 | 기본 없음 | 멀티테넌트면 필수 구현 (현재 공용 MASQUERADE) |
| 벤더 Firecracker | 주의 | live BRANCH 사용 시 포크 유지보수 부담 |
| 실측 검증 | 필요 | 자기 하드웨어에서 `forkd bench` 로 확인 |
| 하드웨어 | 제약 | KVM 되는 베어메탈. 일반 VPS 불가 |

---

## 11. 알려진 한계 (종합)

| 제약 | 내용 |
|---|---|
| x86_64 리눅스 전용 | Ubuntu 22.04+ / KVM 필수. macOS·Windows 직접 불가 |
| Alpha 단계 | 디스크 포맷, API 모양이 1.0 전까지 변경 가능 |
| 단일 노드 | 데몬 1개 = 호스트 1개. 멀티노드 스케줄링 없음 |
| root 권한 필요 | KVM, netns, cgroup 조작 |
| 벤더 Firecracker | v0.4 live BRANCH는 `deeplethe/firecracker` 포크 필요 |
| 보안 감사 미완 | 서드파티 감사 없음. 0.1.4에서 경로 순회 2건 수정 이력 |
| egress 기본 정책 없음 | 현재 공용 MASQUERADE. allow-list는 사용자가 직접 |
| 쿼터 부분적 | `memory.max`만. cpu.max / io.max / pids.max 미지원 |
| 체인 오버헤드 | v0.5 레이어당 +446~692 ms (SHA-256 검증). v0.6에서 최적화 예정 |
| 스냅샷 이식성 | Firecracker 버전 / 호스트 CPU 마이크로아키텍처 간 비호환 |
| 문서 기준 수치 | 이 리포트의 성능 숫자는 저장소 문서의 주장이며 직접 재측정한 값이 아니다 |

---

## 부록 — 게스트/호스트 커널

| 커널 | 누가 고르나 | 버전 | 비고 |
|---|---|---|---|
| **게스트** (각 microVM 내부) | forkd가 제공 | `vmlinux-6.1.141` 고정 | Firecracker CI 검증 이미지. 스냅샷 생성·복원이 동일 게스트 커널에서 수행됨. `scripts/install-guest-kernel.sh` 설치, `forkd doctor` 검증 |
| **호스트** (Firecracker + KVM 실행) | 사용자 머신 | live BRANCH는 ≥ 5.7, 그 외는 KVM 가능한 아무 커널 | `UFFDIO_WRITEPROTECT` on memfd 필요. CI/개발 박스는 6.14 |

---

*이 리포트는 Claude Code 세션에서 `bmshin94/forkd` 저장소를 전수조사하여 작성되었습니다.*
