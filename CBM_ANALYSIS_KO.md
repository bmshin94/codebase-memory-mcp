# codebase-memory-mcp 전수조사 & 활용 전략 정리

> 이 문서는 `codebase-memory-mcp` 프로젝트를 전수조사하고, 설치/사용법, 정체성(MCP/스킬/플러그인),
> AI 에이전트 구축 활용도, 수익화 아이디어, React/PHP 재구현 가능성, 유튜브 콘텐츠 제작 가능성을
> 분석한 결과를 정리한 문서입니다.
>
> - 작성일: 2026-10-07
> - 분석 대상 버전: **v0.11.0**
> - 라이선스: **MIT** (Copyright (c) 2025 DeusData)

## 📎 깃허브 주소

| 구분 | URL |
|---|---|
| **원본 업스트림** | https://github.com/DeusData/codebase-memory-mcp |
| **이 레포 (작업본)** | https://github.com/bmshin94/codebase-memory-mcp |
| 최신 릴리스 | https://github.com/DeusData/codebase-memory-mcp/releases/latest |
| 프로젝트 웹사이트 | https://deusdata.github.io/codebase-memory-mcp/ |
| 이슈 트래커 | https://github.com/DeusData/codebase-memory-mcp/issues |
| AUR 패키지 | https://aur.archlinux.org/packages/codebase-memory-mcp-bin |
| 연구 프리프린트 | https://arxiv.org/abs/2603.27277 (arXiv:2603.27277) |
| MCP 서버 ID | `io.github.DeusData/codebase-memory-mcp` |

---

## 1. 한 줄 정의

> **AI 코딩 에이전트에게 "코드베이스 기억력(지식 그래프)"을 달아주는 네이티브 MCP 서버.**

에이전트가 `grep → read → grep → read` 루프로 코드를 더듬는 대신, 미리 구축된
지식 그래프에 **구조적 질문 한 번**을 던져 답을 얻게 만든다.

```
You: "what calls ProcessOrder?"
Agent → trace_path(function_name="ProcessOrder", direction="inbound")
CBM   → 그래프 쿼리 실행, 구조화된 결과 반환
Agent → 호출 체인을 자연어로 설명
```

**LLM을 내장하지 않는다.** 자연어→쿼리 번역은 이미 대화 중인 에이전트가 담당하므로
추가 API 키·비용·모델 설정이 전혀 필요 없다.

---

## 2. 전수조사 결과 — 폴더 구조

| 폴더 | 내용 | 규모 |
|---|---|---|
| `src/` | 엔진 본체. **순수 C 약 214,000줄** (`.c` 172개 + `.h` 123개) | 9.0M |
| `src/mcp/` | MCP 서버 (JSON-RPC 2.0 / stdio, 17개 툴, 세션 감지, 자동 인덱싱) | — |
| `src/store/` | SQLite 그래프 저장소 (노드/엣지/순회/검색/Louvain 군집화) | — |
| `src/pipeline/` | 다단계 인덱싱 (구조 → 정의 → 호출 → HTTP 링크 → 설정 → 테스트) | — |
| `src/cypher/` | Cypher 렉서 + 파서 + 플래너 + 실행기 (`cypher.c` 5,814줄) | — |
| `src/daemon/` | 계정당 1개 공유 조정 데몬 (IPC, 라이프사이클, 공유 잡/워처) | — |
| `src/semantic/` | 시맨틱(벡터) 검색 | — |
| `src/simhash/` | MinHash / LSH 유사 코드 탐지 | — |
| `src/discover/` | 파일 탐색 (`.gitignore`, `.cbmignore`, symlink 처리) | — |
| `src/watcher/` | 백그라운드 자동 동기화 (git 폴링, 적응형 간격) | — |
| `src/traces/` | 런타임 트레이스 수집 | — |
| `src/cli/` | install / uninstall / update / config (45개 클라이언트 서피스) | — |
| `src/ui/` | 로컬 HTTP 서버 + 검증된 3D UI 애셋 팩 | — |
| `src/foundation/` | 플랫폼 추상화 (스레드, 파일시스템, 로깅, 메모리) | — |
| `internal/cbm/` | **tree-sitter 문법 162개** 벤더링 + AST 추출 엔진 | — |
| `vendored/` | `nomic`(31M, 임베딩 모델), `sqlite3`(9.7M), `mimalloc`, `tre`, `xxhash`, `yyjson` | 42M |
| `graph-ui/` | **React 19 + Three.js(@react-three/fiber) + Vite + Tailwind 4** 3D 시각화 | 652K |
| `tests/` | C / Python / Shell 혼합 테스트. 문서 기준 **8,050~8,060개 / 141 스위트** | 15M |
| `pkg/` | 배포 패키징: npm, PyPI, Homebrew, Scoop, Winget, Chocolatey, AUR, Go, Glama | 504K |
| `tools/` | 커스텀 tree-sitter 문법 6개 (arkts, chialisp, form, magma, tsx, javascript) | — |
| `scripts/` | 빌드 / 린트 / 벤치마크 / 보안감사 / 라이선스게이트 스크립트 40+개 | — |
| `test-infrastructure/` | Docker (alpine, glibc22, mingw, msan) + VM 테스트 래더 | — |
| `docs/` | BENCHMARK, CONFIGURATION, EVALUATION_PLAN, MEASURING_SAVINGS, INDEX_RESOURCE_LIMITS 등 | — |
| `Makefile.cbm` | 빌드 시스템 전체 (83KB) | 83K |
| `flake.nix` | Nix 플레이크 (server / server+UI / frontend 3개 패키지) | — |

---

## 3. 작동 원리 (5단계)

```
[코드베이스]
   │ ① tree-sitter AST 파싱 (162개 언어, 바이너리에 컴파일됨)
   │ ② Hybrid LSP 타입 추론 (12개 언어) — import/제네릭/상속/stdlib 해석
   │ ③ 지식 그래프 구성 (노드 13종 + 엣지 20여 종)
   │ ④ SQLite 영속화 (~/.cache/codebase-memory-mcp/, WAL 모드)
   │ ⑤ MCP(JSON-RPC 2.0 over stdio)로 에이전트에 17개 툴 노출
[Claude Code / Cursor / Codex / ...]
```

### 3-1. Hybrid LSP (핵심 차별 기술)

tree-sitter만으로는 `user.profile.display_name()`이 **어느 클래스의** 메서드인지 모른다.
CBM은 tsserver/typescript-go, pyright, gopls, Roslyn, Eclipse JDT, rust-analyzer의
타입 해석 알고리즘을 **C로 경량 재구현해 바이너리에 내장**했다.
언어 서버 프로세스도, 프로젝트별 설정도, API 키도 없이 IDE의 "Go to Definition" 수준의 해석을 제공한다.

**전체 지원 12개 언어:** Python, TypeScript / JavaScript / JSX / TSX, PHP, C#, Go, C, C++, Java, Kotlin, Rust, Perl

2계층 구조:
1. **tree-sitter 패스** — 162개 언어 전부, 빠른 구문 분석 (정의/호출/import 추출)
2. **Hybrid LSP 패스** — 언어별 타입 인식. import 그래프 + 정의 레지스트리로 호출 엣지 정제.
   Hybrid LSP가 없는 언어는 텍스트 기반 해석으로 폴백 → **항상 어떤 답은 나온다**

### 3-2. 그래프 데이터 모델

**노드 라벨 (13종)**
`Project`, `Package`, `Folder`, `File`, `Module`, `Class`, `Function`, `Method`,
`Interface`, `Enum`, `Type`, `Route`(REST 엔드포인트), `Resource`(K8s 리소스)

**주요 엣지 타입**

| 엣지 | 의미 |
|---|---|
| `CALLS` | 호출 사이트에서 실제로 호출됨 |
| `CALL_REFERENCE` | 참조 사이트에서 사용되며 정확히 1개 타깃으로 해석됨 |
| `USAGE` | 식별자가 사용되지만 유일한 타깃이 증명되지 않음 (모호/복합 표현) |
| `IMPORTS`, `DEFINES`, `DEFINES_METHOD`, `IMPLEMENTS`, `INHERITS` | 구조 관계 |
| `HTTP_CALLS`, `ASYNC_CALLS` | **크로스 서비스** 호출 연결 |
| `SPAWNS` | 다른 프로그램 실행 (`subprocess.run`, `exec.Command`, `Process.Start` 등) |
| `EMITS`, `LISTENS_ON` | 이벤트 발행 / 수신 (Socket.IO, EventEmitter, 범용 pub-sub) |
| `DATA_FLOWS` | arg→param 매핑 + 필드 접근 체인 |
| `SIMILAR_TO` | MinHash + LSH 근사 복제 탐지 (Jaccard 점수) |
| `SEMANTICALLY_RELATED` | 어휘 불일치, 동일 언어, 점수 ≥ 0.80 |
| `FILE_CHANGES_WITH` | 함께 수정되는 경향이 있는 파일 (git 히스토리) |
| `HANDLES`, `CONFIGURES`, `REFERENCES_FILE`, `WRITES`, `MEMBER_OF`, `TESTS`, `USES_TYPE` | 기타 |

**정규화 이름(Qualified Name):** `<project>.<path_parts>.<name>`
→ `get_code_snippet`이 사용. `search_graph`로 먼저 발견해야 함.

---

## 4. MCP 툴 17개

### 인덱싱 (5)

| 툴 | 설명 |
|---|---|
| `index_repository` | 레포 인덱싱. 기본은 동기. `async: true`로 데몬에서 시작 후 즉시 반환, `status: true`로 폴링 |
| `list_projects` | 인덱싱된 프로젝트 목록 + 노드/엣지 수 |
| `delete_project` | 프로젝트 및 그래프 데이터 삭제 |
| `index_status` | 인덱싱 상태 확인 |
| `check_index_coverage` | 특정 경로/범위가 인덱싱되고 신선한지 확인. **깨끗한 결과 = "기록된 누락 없음", 완전성 증명은 아님** |

> **긴 인덱싱 주의:** 일부 IDE 클라이언트는 툴 호출 데드라인이 있어 동기 호출이 취소된다.
> 취소되면 데몬이 인덱싱을 중단하므로 같은 동기 호출을 재시도해도 끝나지 않는다.
> `async: true` → `status: true` 폴링 패턴을 사용할 것.
> (`async`와 `status`는 상호 배타. `async`는 `cross-repo-intelligence`에 적용 안 됨)

### 쿼리 (12)

| 툴 | 설명 |
|---|---|
| `search_graph` | **구조 + BM25 + 시맨틱** 검색 통합. 구조 행은 `offset`/`limit`, 시맨틱 행은 `semantic_offset`/`semantic_limit`로 독립 페이징 |
| `trace_path` | BFS 호출 추적 (`inbound`/`outbound`/`both`, depth 1~5). 별칭 `trace_call_path` |
| `detect_changes` | git diff → 영향 심볼 + 블래스트 반경 + 리스크 분류 |
| `query_graph` | Cypher 유사 쿼리 실행 (읽기 전용) |
| `get_graph_schema` | 노드/엣지 수, 관계 패턴, 라벨별 속성 정의. **먼저 실행 권장** |
| `compare_graphs` | 두 스냅샷 비교 (노드/엣지 추가·삭제) |
| `get_code_snippet` | 정규화 이름으로 함수 소스 읽기 |
| `get_file_outline` | 파일 1개의 선언 개요 (소스 순서, 라벨 필터, 페이징) |
| `get_architecture` | 개요 한 방: 언어 / 패키지 / 엔트리포인트 / 라우트 / 핫스팟 / 경계 / 레이어 / 클러스터 / ADR |
| `search_code` | 인덱싱된 파일 내 grep 유사 텍스트 검색 |
| `manage_adr` | 아키텍처 결정 기록 CRUD. `set_sections`는 명시한 섹션만 바이트 단위로 교체 |
| `ingest_traces` | 런타임 트레이스 수집 → `HTTP_CALLS` 엣지 검증 |

### 지원 Cypher (openCypher 읽기 서브셋)

- **절**: `MATCH`, `OPTIONAL MATCH`, 다중 `MATCH`, `WHERE`, `WITH`(+`WITH … WHERE`), `RETURN`, `ORDER BY`, `SKIP`, `LIMIT`, `DISTINCT`, `UNWIND`, `UNION`/`UNION ALL`, `CASE`
- **패턴**: 라벨 노드, 라벨 대안 `(n:A|B)`, 관계 타입/방향, 가변 길이 경로 `[*1..3]`, 인라인 속성 맵
- **WHERE**: 비교 연산자, `AND/OR/XOR/NOT`, `IN`, `CONTAINS`, `STARTS WITH`, `ENDS WITH`, `IS [NOT] NULL`, 정규식 `=~`, 라벨 테스트 `n:Label`, `EXISTS { (n)-[:TYPE]->() }`
- **집계**: `count`(+`DISTINCT`), `sum`, `avg`, `min`, `max`, `collect`
- **함수**: `labels`, `type`, `id`, `keys`, `properties`, `toLower/toUpper/toString/toInteger/toFloat/toBoolean`, `size`, `length`, `trim/ltrim/rtrim`, `reverse`, `coalesce`, `substring`, `replace`, `left`, `right`
- 서브셋 밖은 **빈 결과가 아니라 명확한 `unsupported …` 에러**로 실패 (쓰기/`MERGE`/`CALL`, 리스트·맵 리터럴, 컴프리헨션, 경로 함수, 파라미터 등)

**데드코드 탐지 예시**
```cypher
MATCH (f:Function) WHERE NOT EXISTS { (f)<-[:CALLS]-() } RETURN f.name
```

---

## 5. 성능 (Apple M3 Pro, 프로젝트 문서 기준)

| 작업 | 시간 | 비고 |
|---|---|---|
| **리눅스 커널 풀 인덱싱** | **3분** | 28M LOC, 75K 파일 → 노드 4.81M, 엣지 7.72M |
| 리눅스 커널 fast 인덱싱 | 1분 12초 | 노드 1.88M |
| Django 풀 인덱싱 | ~6초 | 노드 49K, 엣지 196K |
| Cypher 쿼리 | **<1ms** | 관계 순회 |
| 이름 검색(정규식) | <10ms | SQL LIKE 선필터 |
| 데드코드 탐지 | ~150ms | 전체 그래프 스캔 |
| 호출 경로 추적(depth 5) | <10ms | BFS |

**RAM-first 파이프라인:** LZ4 HC 압축 읽기 → 인메모리 SQLite → 종료 시 1회 덤프. 인덱싱 후 메모리 OS 반환.

**토큰 효율:** 5개 구조 쿼리 ~3,400 토큰 vs 파일별 grep 탐색 ~412,000 토큰 → **99.2% 절감**

> ⚠️ 위 수치는 모두 **프로젝트 자체 주장**이다. 직접 검증한 값이 아니다.
> 본인 환경 실측 방법은 `docs/MEASURING_SAVINGS.md` 참고.
> 정확한 재현에는 원본 입력과 raw 아티팩트가 필요하다.

**언어 지원 등급 (64개 실제 오픈소스 레포 벤치마크)**
- **Excellent (≥90%)**: Lua, Kotlin, C++, Perl, Objective-C, Groovy, C, Bash, Zig, Swift, CSS, YAML, TOML, HTML, SCSS, HCL, Dockerfile
- **Good (75~89%)**: Python, TypeScript, TSX, Go, Rust, Java, R, Dart, JavaScript, Erlang, Elixir, Scala, Ruby, PHP, C#, SQL
- **Functional (<75%)**: OCaml, Haskell
- 그 외 100여 개 언어는 지원되나 미벤치마크

---

## 6. 설치 및 사용법

### 6-1. 설치 (7가지 경로)

**① 원라인 설치 (권장)**
```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash
```
```powershell
# Windows PowerShell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.ps1 -OutFile install.ps1
notepad install.ps1        # 실행 전 확인 권장
Unblock-File .\install.ps1
.\install.ps1
```
- 실행 정책 에러 시: `Set-ExecutionPolicy -Scope Process Bypass`
- 옵션: `--skip-config`(바이너리만), `--dir=<경로>`

**② 패키지 매니저**
```bash
npm install -g codebase-memory-mcp
pip install codebase-memory-mcp
brew install ...            # Homebrew
scoop / choco / winget      # Windows
yay -S codebase-memory-mcp-bin   # Arch AUR
go install ...
```

**③ Nix 플레이크**
```bash
nix run github:DeusData/codebase-memory-mcp
nix run github:DeusData/codebase-memory-mcp#codebase-memory-mcp-ui -- --ui=true --port=9749
nix build github:DeusData/codebase-memory-mcp#codebase-memory-mcp-ui
```

**④ 수동 다운로드** — 릴리스에서 플랫폼별 아카이브 (`checksums.txt` 동봉)

| 플랫폼 | 아카이브 |
|---|---|
| macOS (Apple Silicon) | `codebase-memory-mcp-darwin-arm64.tar.gz` |
| macOS (Intel) | `codebase-memory-mcp-darwin-amd64.tar.gz` |
| Linux (x86_64) | `codebase-memory-mcp-linux-amd64.tar.gz` |
| Linux (ARM64) | `codebase-memory-mcp-linux-arm64.tar.gz` |
| Windows (x86_64) | `codebase-memory-mcp-windows-amd64.zip` |

**⑤ 에이전트에 시키기**
```
"이 MCP 서버 설치해줘: https://github.com/DeusData/codebase-memory-mcp"
```

**⑥ 소스 빌드** (필요: gcc/clang, g++, zlib, git)
```bash
git clone https://github.com/DeusData/codebase-memory-mcp.git
cd codebase-memory-mcp
scripts/build.sh --with-ui    # 배포와 동일한 구성 (graph UI 내장)
scripts/build.sh              # UI 없이 (개발용)
# 결과물: build/c/codebase-memory-mcp

scripts/test.sh                       # 전체 테스트 (CI와 동일 엔트리)
scripts/test.sh --suites <name>       # 단일 스위트
build/c/test-runner --list-suites     # 스위트 목록
```

**⑦ 수동 MCP 설정** (`~/.claude.json` 또는 프로젝트 `.mcp.json`)
```json
{
  "mcpServers": {
    "codebase-memory-mcp": {
      "command": "/절대/경로/codebase-memory-mcp",
      "args": []
    }
  }
}
```
→ 에이전트 재시작 후 `/mcp`에서 17개 툴 확인

### 6-2. 설정

```bash
codebase-memory-mcp config list
codebase-memory-mcp config set auto_index true            # 세션 시작 시 자동 인덱싱
codebase-memory-mcp config set auto_index_limit 50000     # 자동 인덱싱 파일 한도
codebase-memory-mcp config set auto_watch false           # 워처 등록 안 함 (세션별)
codebase-memory-mcp config set watcher_enabled false      # 워처 스레드 자체 비활성 (데몬 시작 시 1회 읽힘)
codebase-memory-mcp config set watch_non_git true         # git 아닌 루트도 폴링
codebase-memory-mcp config set index_max_files 250000     # 인덱스당 파일 한도 (기본 off)
codebase-memory-mcp config set index_max_source_mb 16384  # 인덱스당 소스 크기 한도 (기본 off)
codebase-memory-mcp config reset auto_index
```

> `watcher_enabled`는 데몬 시작 시 1회만 읽힌다. 변경 후 `codebase-memory-mcp daemon stop` 필요.
> `index_max_*` 초과 시 부분 그래프를 퍼블리시하지 않고 인덱싱 전체가 실패하며, 기존 인덱스는 보존된다.

**커스텀 확장자 매핑** (`.codebase-memory.json` 또는 `~/.config/codebase-memory-mcp/config.json`)
```json
{"extra_extensions": {".blade.php": "php", ".mjs": "javascript", ".twig": "html"}}
```
확장자는 `.`으로 시작해야 하고, 언어명은 대소문자 무시 매칭. 프로젝트 설정이 전역 설정을 덮어쓴다.

### 6-3. 사용법 A — 에이전트에서 (자연어)

```
"이 프로젝트 인덱싱해줘"
"구조 전체 설명해줘"
"UserService.login 호출하는 데 다 찾아줘"
"안 쓰는 함수 찾아줘"
"내 변경사항 영향 범위 알려줘"
```

### 6-4. 사용법 B — CLI (17개 툴 전부 호출 가능)

```bash
codebase-memory-mcp cli index_repository --repo-path /path/to/repo
codebase-memory-mcp cli list_projects
codebase-memory-mcp cli search_graph --project my-app --name-pattern '.*Handler.*' --label Function
codebase-memory-mcp cli trace_path --project my-app --function-name Search --direction both
codebase-memory-mcp cli query_graph --project my-app --query 'MATCH (f:Function) RETURN f.name LIMIT 5'

# 자동화용 JSON
codebase-memory-mcp cli list_projects --format json --detail stats | jq '.projects[].name'
# 진행률 강제 표시 / 조용히
codebase-memory-mcp cli --progress index_repository --repo-path /path/to/repo
codebase-memory-mcp cli --quiet list_projects --format json
# 플래그 확인
codebase-memory-mcp cli <tool> --help
```

- stdout = 명령 결과 전용, stderr = 진행률/로그
- 읽기 툴은 기본 compact tree, `--format json`은 payload JSON, 외부 `--json`은 전체 MCP 엔벨로프
- JSON 인자는 stdin 파이프 / `--args-file` 가능 (인라인 JSON은 하위호환용 deprecated)
- 그래프 변경 명령은 OS 기반 프로젝트별 락으로 직렬화 (다른 프로젝트는 병행 진행)

### 6-5. 사용법 C — 3D 그래프 UI

```bash
codebase-memory-mcp --ui=true --port=9749
# http://localhost:9749
```
> 손으로 실행하면 stdin이 닫히는 순간 종료된다(MCP 정상 동작).
> 테스트: `sleep infinity | codebase-memory-mcp --ui=true --port=9749`
> UI는 공유 조정 데몬이 소유하므로 동시 세션이 HTTP 서버를 중복 기동하지 않는다.

### 6-6. 업데이트 / 삭제

```bash
# 업데이트는 install 스크립트로 실행 (실행 중 바이너리는 자기 이미지를 교체할 수 없음)
bash "<install-dir>/install.sh"                                   # macOS/Linux
powershell -ExecutionPolicy Bypass -File "<install-dir>\install.ps1"  # Windows
npm install -g codebase-memory-mcp@latest     # npm 설치자
pip install -U codebase-memory-mcp            # pip 설치자

# 삭제 (인덱스는 기본 보존)
codebase-memory-mcp uninstall
codebase-memory-mcp uninstall -y --delete-indexes
codebase-memory-mcp uninstall --dry-run --delete-indexes
```
- `--delete-indexes`는 명시적 동의이며 `--no`보다 우선. `--dry-run`은 항상 파일 보존
- 바이너리 옆 install 스크립트는 **삭제하지 않고 경로만 보고**한다 (소유 증명 불가 파일은 지우지 않는 원칙)

### 6-7. 문제 해결

| 문제 | 해결 |
|---|---|
| `/mcp`에 서버가 없음 | 경로를 절대경로로 확인 → 에이전트 재시작. 테스트: `echo '{}' \| /path/to/binary` |
| `index_repository` 실패 | `repo_path`를 절대경로로 |
| 클라이언트에서 인덱싱 타임아웃 | `async: true`로 시작 후 `status: true` 폴링 |
| `trace_path` 결과 0개 | `search_graph(name_pattern=".*부분이름.*")`로 정확한 이름 먼저 확인 |
| 다른 프로젝트 결과 혼입 | `project="이름"` 지정 (`list_projects`로 이름 확인) |
| 설치 후 명령을 찾지 못함 | `export PATH="$HOME/.local/bin:$PATH"` |
| UI가 안 열림 | `--ui=true` 여부 + 포트 9749 확인 |

### 6-8. 진단 (메모리/성능 이슈 리포트용)

CBM은 100% 로컬이고 텔레메트리를 전혀 수집하지 않으므로, 재현 불가 이슈는 사용자가 직접 수집해야 한다.

```bash
export CBM_DIAGNOSTICS=1   # 첫 데몬 세션 시작 전에 설정
```
- `trajectory.ndjson` — 5초마다 `rss`, `committed`, `peak_*`, `page_faults`, `fd`, `queries` 기록.
  **메모리/누수 리포트에 필요한 파일.** 종료 후에도 디스크에 남고 ~8MB 초과 시 로테이트
- `snapshot.json` — 최신 스냅샷만. 정상 종료 시 삭제
- 경로는 `${CBM_CACHE_DIR}/logs/cbm-daemon.log`의 `diagnostics.start` 이벤트에 기록
- 공유 가능: NDJSON에는 소스코드나 쿼리 텍스트가 없고 리소스 카운터만 들어있다

---

## 7. 정체성 — 플러그인? 스킬? MCP?

### 결론: **MCP 서버가 본체.** 설치 시 스킬 + 훅 + 서브에이전트까지 부가로 깔아준다.

| 분류 | 해당 | 설명 |
|---|:---:|---|
| **MCP 서버** | ✅ **본체** | `server.json`에 MCP 공식 스키마 선언. ID `io.github.DeusData/codebase-memory-mcp`. 17개 툴, stdio, JSON-RPC 2.0 |
| **Skill** | ✅ 부가 | `install`이 `SKILL.md`를 각 클라이언트 스킬 경로에 설치 |
| **Hooks** | ✅ 부가 | `SessionStart`, `SubagentStart`, `PreToolUse`(Grep/Glob/Bash), `PostToolUse`(Read). **fail-open & 컨텍스트 전용 — 호출을 차단/대체하지 않음** |
| **Subagents** | ✅ 부가 | **Scout / Verify / Auditor** 3티어 자동 생성 |
| Claude Code 플러그인 | ❌ | `.claude-plugin/` 형식이 아님 |
| VSCode 확장 | ❌ | 아님. 다만 VS Code `mcp.json`에 MCP 등록은 해줌 |
| 라이브러리 / SDK | ❌ | import 대상이 아니라 독립 실행 바이너리 |

### 3티어 에이전트

| 티어 | 성격 |
|---|---|
| **Scout (Tier 1)** | 3~4회의 좁은 호출로 빠른 **잠정** 탐색. **"없음" 주장·완전 영향분석·데드코드 주장 금지** |
| **Verify (Tier 2, 기본)** | 과제 지향 그래프 근거 + 정확한 소스 확인 + 인용 파일 전부 경로 커버리지 + 부정 주장 전 범위 커버리지 |
| **Auditor (Tier 3)** | 범위 한정 + 현재 인덱스 세대 + 완전 페이지네이션 + 광범위 관계 확인 + 미해결 한계 명시 |

모든 티어가 근거 경로에 대해 `check_index_coverage`를 배치 실행하고, 플래그된 범위·스킵·제외 파일은 직접 읽는다.
**"커버리지 깨끗함"은 "기록된 누락 없음"일 뿐 완전성 증명이 아니다**라고 명시.
업데이트는 **바이트 단위로 동일한 기존 Verify 정의만** 마이그레이션하고 사용자가 수정한 에이전트는 덮어쓰지 않는다.

### 지원 클라이언트 45개 서피스 (39 자동 감지 + 6 조건부/명시)

**자동 감지:** Claude Code, Codex CLI, Gemini CLI, Zed, OpenCode, Antigravity, Aider, KiloCode,
VS Code, Cursor, Windsurf, Augment/Auggie, OpenClaw, Kiro, Junie, Hermes, OpenHands, Cline, Warp,
Qwen Code, GitHub Copilot CLI, Factory Droid, Crush, Goose, Mistral Vibe, Grok Build, Qoder CLI,
Kimi Code CLI, GitLab Duo CLI, Rovo Dev CLI, Amp, Devin CLI/Local, Tabnine, Amazon Q Developer IDE,
CodeBuddy Code CLI, IBM Bob Shell, Pochi, Pi, Oh My Pi (omp)

**조건부(기존 설정 파일이 있을 때만):** Continue/cn, Visual Studio(Windows), TRAE, Roo Code,
IBM Bob IDE, Sourcegraph Cody(명시적 opt-in)

**의도적으로 자동 설치에서 제외:** Qodo(UI/엔터프라이즈 허용목록), Warp MCP(Warp Drive/UI 관리),
JetBrains AI Assistant/ACP(IDE 관리), GitHub Copilot coding agent / Jules / CodeRabbit(클라우드 관리),
Replit(원격 서비스), BLACKBOX AI(안정 스키마 미문서), Plandex(안전한 전역 레지스트리 없음), SWE-agent(명시적 YAML)

> **보안 설계 평가:** 실험적 기능 플래그, 플러그인 활성화, YOLO 모드, 전역 권한 우회,
> 서드파티 지침 신뢰를 **절대 건드리지 않는다.** 문서화된 플랫폼/마커/기존 설정 경로가
> 타깃 활성을 증명할 때만 기록한다. 훅은 전부 fail-open. 매우 보수적이고 좋은 설계.

---

## 8. API 토큰 필요 여부

### 결론: **단 하나도 필요 없다.** 이것이 이 프로젝트의 최대 셀링포인트.

| 항목 | 일반 경쟁 도구 | CBM |
|---|---|---|
| LLM API 키 | 필요 (자연어→쿼리 번역) | ❌ 불필요 — 에이전트가 번역기 |
| 임베딩 API 키 | 필요 (OpenAI/Cohere) | ❌ 불필요 — `nomic-embed-code`(40K 토큰, 768d int8) 바이너리 내장 |
| Ollama / 로컬 LLM | 필요 | ❌ 불필요 |
| Docker | 필요 | ❌ 불필요 |
| Neo4j / 벡터DB 서버 | 필요 | ❌ 불필요 — SQLite |
| 언어 런타임(Node/Python) | 필요 | ❌ 불필요 — 순수 C |
| 클라우드 계정 | 필요 | ❌ 불필요 |

**시맨틱 검색 11-신호 결합 점수:** TF-IDF, RRI, API/Type/Decorator 시그니처, AST 프로필,
데이터 흐름, Halstead-lite, MinHash, 모듈 근접도, 그래프 확산

### 프라이버시
- **100% 로컬 처리.** 코드가 기기를 떠나지 않음
- **텔레메트리 수집 0.** 백그라운드 버전 체크조차 안 함. 릴리스 아카이브에 다운로드 URL 자체가 없음
- 이슈 리포트 시 사용자가 직접 `CBM_DIAGNOSTICS=1`로 수집해 보내야 함 (서버 측 데이터가 애초에 없음)

### 환경 변수 (전부 선택사항, 인증 키 아님)

| 변수 | 기본값 | 용도 |
|---|---|---|
| `CBM_ALLOWED_ROOT` | 없음 | 인덱싱을 이 디렉터리 하위로 제한. 멀티테넌트/비신뢰 호출자 환경에 유용. UI의 `POST /api/index`에도 적용 |
| `CBM_CACHE_DIR` | `~/.cache/codebase-memory-mcp` | DB 저장 위치. 계정당 canonical 루트 1개만 허용 |
| `CBM_DIAGNOSTICS` | `false` | 진단 수집 |
| `CBM_DOWNLOAD_URL` | GitHub releases | 셀프호스팅 다운로드 URL |
| `CBM_LOG_LEVEL` | 역할별 | `debug`/`info`/`warn`/`error`/`none` (또는 0~4) |
| `CBM_WORKERS` | 자동 감지 | 병렬 워커 수 1~256. 컨테이너에서 cgroup 쿼터와 호스트 CPU 불일치 시 유용 |
| `CBM_MEM_BUDGET_MB` | 자동 감지 | 인메모리 그래프 예산 상한(MiB) |
| `CBM_DUMP_VERIFY_MIN_RATIO` | `0.5` | 영속 노드 수가 커밋 노드 수의 이 비율 미만이면 `status:"degraded"` 반환 |

> 데몬 소유 환경(진단·로깅·리소스 한계)은 **데몬을 시작한 첫 세션에서 캡처**된다.
> 이후 세션은 그 값을 교체할 수 없다. 변경하려면 모든 세션 종료 후 재시작.

---

## 9. AI 에이전트 구축에 도움되는가

### 결론: 매우 크게 도움된다. 이 프로젝트 자체가 "AI 에이전트용 인프라"다.

**① 토큰/비용 경제성** — 컨텍스트 윈도우가 곧 비용이고 성능이다. grep/read 루프가 토큰을
다 먹으면 에이전트 품질이 떨어진다. 상용 코딩 에이전트를 만든다면 필수 레이어.

**② "Code RAG" 레퍼런스 아키텍처** — 흔한 방식(파일 분할 → 임베딩 → 벡터DB → 유사도)은
코드의 **구조를 무시**해 품질이 낮다. CBM은 AST 그래프 + 타입 추론 + 벡터를 결합한다.
Code RAG를 만들 거라면 이 아키텍처를 그대로 참고해야 한다.

**③ 멀티에이전트 설계 패턴 교재** — Scout/Verify/Auditor 3티어 + 프로세스 툴 프로필
(Scout = 7개 빠른 검사 툴, Analysis = 11개 / **positive allowlist**, 미래·변경 툴은 명시적 리뷰 전까지 미노출)
+ 부모→자식 핸드오프 컨트랙트(프로젝트, 세대, 페이지네이션 상태, 그래프 근거, 커버리지 결과 전달).

**④ 에이전트 환각 방지 장치** — "커버리지 깨끗함 ≠ 완전성 증명", Scout의 부정 주장 금지,
`unsupported …` 명확 실패(빈 결과 아님). **에이전트가 확신 없는 걸 단정하지 않게 만드는** 프롬프트/API 설계의 모범 사례.

**⑤ 프로덕션 MCP 서버 구현 레퍼런스**
- JSON-RPC 2.0 over stdio, **stdout은 MCP 전용 / 로그는 stderr**
- 긴 작업용 `async` + `status` 폴링 (클라이언트 데드라인 회피)
- `max_output_tokens` 기반 **의미 보존 truncation** — 바이트를 임의로 자르지 않고,
  랭킹된 그래프 행을 raw grep 행보다 우선 보존, 생략 시 총계·`has_more`·**단조 증가 커서** 제공.
  첫 행조차 못 들어가면 더 큰 예산을 요청하고 self-looping 커서를 내보내지 않음.
  모델 중립 사이징(요청 토큰당 UTF-8 4바이트 상한)
- 토큰 절약용 `_refs` 압축 테이블 — 렌더된 테이블이 **15% 이상 + 64바이트 이상** 작아지고
  모델 중립 토큰 형태 프록시가 **1% 이상** 개선될 때만 활성. 두 게이트 모두 결정론적
- UTF-8 안전성 — 깨진 바이트는 `@bytes:<hex>`로 가역 인코딩, 리터럴 충돌은 `@utf8:` 접두사

**⑥ 에이전트 평가(Eval) 방법론** — `docs/EVALUATION_PLAN.md`, `docs/MEASURING_SAVINGS.md`,
`docs/BENCHMARK.md`. arXiv 프리프린트에 31개 실제 레포 평가 (답변 품질 83%, 토큰 10배 절감, 툴 호출 2.1배 감소 주장)

### 내 에이전트에 붙이는 3가지 방법
```
① MCP로 붙이기   — 에이전트가 MCP 클라이언트면 설정 한 줄
② CLI로 붙이기   — child_process.spawn / subprocess + --format json (언어 무관)
③ 아키텍처 차용 — 그래프 스키마 + 티어 설계 + 쿼리 패턴만 내 스택으로 재구현
```

### 한계
- CBM은 **구조만** 안다. "왜 이렇게 됐는지", "비즈니스 로직이 맞는지"는 LLM 몫
- **인덱스 신선도 관리 필수.** 오래된 그래프를 믿으면 사고. `index_status` / `check_index_coverage` 중요
- OCaml, Haskell은 75% 미달. 미벤치마크 언어 다수
- **C 코드베이스라 확장 난이도 높다.** 기능 추가하려면 C를 만져야 함

---

## 10. React / PHP로 만들 수 있는가

### 결론: "엔진 전체 복제"는 비현실적. "실용적 축소판"은 충분히 가능.

| 구성요소 | React/TS | PHP | 비고 |
|---|:---:|:---:|---|
| MCP 서버 (JSON-RPC/stdio) | 매우 쉬움 | 쉬움 | TS는 공식 SDK 존재 |
| tree-sitter 파싱 | 쉬움 | 보통 | `web-tree-sitter`(WASM) / `nikic/PHP-Parser` 또는 FFI |
| 그래프 저장 (SQLite) | 쉬움 | 쉬움 | better-sqlite3 / PDO |
| 구조 검색 + BM25 | 쉬움 | 쉬움 | SQLite FTS5 그대로 |
| 호출 그래프 추출 | 보통 | 보통 | 언어별 수작업 |
| 3D 그래프 UI | **이미 React** | 부적합 | graph-ui = React 19 + Three.js |
| Cypher 파서 | 어려움 | 어려움 | 5,814줄. 서브셋만 하면 가능 |
| Hybrid LSP 타입 추론 | 매우 어려움 | 매우 어려움 | 12개 언어 타입 시스템 재구현 |
| 시맨틱 벡터 검색 | 보통 | 어려움 | transformers.js / 외부 임베딩 API |
| 성능 (커널 3분) | 불가 | 불가 | GC 언어로는 C 성능 추격 불가 |
| 162개 언어 | 비현실적 | 비현실적 | 초기 목표로 부적합 |

### Plan A — TypeScript/Node "미니 CBM" (권장)

```
1주차  MCP 서버 뼈대: @modelcontextprotocol/sdk, stdio, 툴 3개 스텁
2주차  인덱서 (언어 2개만: TypeScript + Python)
         web-tree-sitter 파싱 → 함수/클래스/import 추출 → better-sqlite3 nodes/edges
3주차  쿼리 툴: search_graph(정규식 + FTS5 BM25), trace_path(재귀 CTE BFS), get_architecture(집계)
4주차  영향 분석: simple-git diff → 변경 라인 → 심볼 매핑 → 역추적
5주차  3D UI: react-force-graph-3d 또는 @react-three/fiber
6주차  (선택) 시맨틱 검색: transformers.js 로컬 임베딩 또는 외부 API
```

**핵심 SQL — BFS를 재귀 CTE로**
```sql
-- trace_path(inbound): 이 함수를 호출하는 애들, depth 3까지
WITH RECURSIVE callers(id, name, depth) AS (
  SELECT id, name, 0 FROM nodes WHERE name = :target AND label = 'Function'
  UNION
  SELECT n.id, n.name, c.depth + 1
  FROM callers c
  JOIN edges e ON e.dst_id = c.id AND e.type = 'CALLS'
  JOIN nodes n ON n.id = e.src_id
  WHERE c.depth < 3
)
SELECT DISTINCT name, depth FROM callers WHERE depth > 0 ORDER BY depth;
```

예상 투입: 개발자 1명 + AI 어시스턴트 → **4~6주면 실용 수준**

### Plan B — PHP "라라벨/워드프레스 전문 분석기" (틈새 공략)

PHP로 162개 언어는 비효율. **PHP 생태계 전문**으로 좁히면 경쟁이 거의 없다.

```
파싱: nikic/PHP-Parser (PHP를 PHP로 파싱, 품질 최상)
저장: PDO + SQLite / MySQL
MCP: php://stdin 읽고 JSON-RPC 응답
UI:  Laravel Blade + Cytoscape.js (2D 그래프)
```

**PHP 전용 킬러 기능 (CBM도 완벽히 못 하는 것)**
- 라라벨 라우트 ↔ 컨트롤러 ↔ 모델 ↔ 마이그레이션 전체 체인 추적
- Eloquent 관계(hasMany/belongsTo) 그래프 + 자동 ERD
- Blade 템플릿 ↔ 컨트롤러 변수 흐름
- **워드프레스 훅/필터(`add_action`/`apply_filters`) 추적** — 레거시 WP 분석의 금광
- PSR-4 오토로드 ↔ 실제 클래스 매핑
- PHP 5.6 → 8.x 마이그레이션 리포트, 플러그인 충돌 탐지

### Plan C — 하이브리드 (가장 영리한 길, 최우선 추천)

**엔진은 CBM을 쓰고, 그 위에 React/PHP로 레이어를 얹는다.**

```
┌──────────────────────────────────────────────┐
│  직접 만드는 레이어 (React / PHP / Laravel)    │
│  · 팀용 대시보드 (기술부채, 핫스팟 랭킹)        │
│  · PR별 리스크 리포트 자동 생성                 │
│  · 아키텍처 문서/다이어그램 자동 생성            │
│  · 슬랙/팀즈 알림 (고위험 변경 감지)            │
│  · 멀티 레포 관리 / 권한 / 조직 뷰              │
│  · 코드 리뷰 코멘트 자동 생성                   │
└────────────────┬─────────────────────────────┘
                 │ CLI --format json  또는  MCP
┌────────────────▼─────────────────────────────┐
│  codebase-memory-mcp (C 엔진, MIT)            │
└──────────────────────────────────────────────┘
```

장점: 가장 어려운 부분(파싱/타입추론/성능)을 무료로 해결, MIT라 상업적 이용 가능,
UI/UX와 비즈니스 로직에만 집중, 수 주 내 실제 제품 가능.

**Node/TS에서 호출**
```ts
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
const run = promisify(execFile);

async function cbm(tool: string, args: string[]) {
  const { stdout } = await run('codebase-memory-mcp',
    ['cli', tool, ...args, '--format', 'json', '--quiet']);
  return JSON.parse(stdout);
}
const arch = await cbm('get_architecture', ['--project', 'my-app']);
const risk = await cbm('detect_changes',   ['--project', 'my-app']);
```

**PHP에서 호출**
```php
$json = shell_exec('codebase-memory-mcp cli get_architecture '
                 . '--project my-app --format json --quiet');
$arch = json_decode($json, true);
```

---

## 11. 유튜브 강의 영상 제작 가능성

### 결론: 가능한 수준을 넘어 소재로서 매우 우수하다.

**왜 좋은가**
1. **비주얼이 압도적** — 3D 그래프 화면. 개발 콘텐츠의 최대 약점("볼 게 없음")이 해결됨
2. **숫자가 자극적** — "토큰 99.2% 절감", "리눅스 커널 3분", "쿼리 1ms"
3. **타이밍** — MCP는 현재 최대 화두. 한국어 MCP 심화 콘텐츠는 희소
4. **레이어가 많음** — 입문(설치) ~ 고급(C 코드 리딩). 시리즈 설계 용이
5. **희소성** — 순수 C MCP 서버, 그래프 기반 코드 지능. 한국어 자료 거의 없음
6. **공신력** — arXiv 프리프린트, SLSA Level 3, OpenSSF Scorecard, VirusTotal

### 시리즈 기획안 (12편)

| EP | 제목 | 길이 | 목적 |
|---|---|---|---|
| 1 | 클로드 코드 토큰 99% 절약하는 MCP 서버 (무료, API키 없음) | 8~10분 | 훅 / 조회수 |
| 2 | 내 코드베이스를 3D로 보기 | 6~8분 | 비주얼 / Shorts 소재 |
| 3 | MCP가 뭔데 다들 난리야? | 10~12분 | 검색 유입 (상록수) |
| 4 | 처음 보는 레거시 프로젝트 10분에 파악하기 | 12~15분 | 공감 / 실전 |
| 5 | 리팩토링 전에 이거 돌려보세요 (블래스트 반경) | 10~12분 | 실무자 저장 |
| 6 | 안 쓰는 코드 1,000줄 지우기 | 10분 | 카타르시스 |
| 7 | Cypher 쿼리로 내 코드에 질문하기 | 12~15분 | 차별화 |
| 8 | MSA 서비스 간 호출 추적하기 | 15분 | 시니어/아키텍트 |
| 9 | C로 MCP 서버 만든 사람들 (21만 줄 코드 리딩) | 20분+ | 전문성/신뢰 |
| **10** | **나도 MCP 서버 만들어보자 (TypeScript 실습)** | **60분** | **핵심. 유료 강의의 씨앗** |
| 11 | 라라벨 프로젝트 전용 코드 분석기 만들기 (PHP) | 40분 | 한국 PHP 시장 |
| 12 | 팀에 도입하기 — 공유 그래프 아티팩트 & CI 연동 | 20분 | B2B 리드 |

**EP1 구성 예시**
```
0:00  훅 — 토큰 사용량 Before/After 화면 분할
0:40  문제 정의: 왜 에이전트가 grep으로 삽질하나
2:00  해결책 소개 + 3D 그래프 티저
3:00  설치 (원라인, 실시간)
5:00  "이 프로젝트 인덱싱해줘" → 실제 작업 시연
8:00  토큰 비교 결과 공개
9:30  다음 편 예고
```

### 제작 실무

**녹화 셋업**
- 화면 1080p 60fps 이상 (3D 그래프는 fps 중요) 또는 1440p
- 터미널 폰트 18pt 이상 (모바일 가독성), 배경은 3D UI와 톤 통일
- 인덱싱 대기는 2~4배속 + 타이머 오버레이. "3분" 같은 수치는 실제 타이머 노출(신뢰도)

**비주얼 무기**
- 3D 그래프 회전 샷 (인트로 B롤용 10초 따로 녹화)
- 토큰 사용량 Before/After 막대그래프 애니메이션
- `trace_path` 결과를 그래프에서 하이라이트
- `detect_changes` 리스크 등급 색상(빨강/노랑/초록)

### 반드시 지킬 것

| 항목 | 지침 |
|---|---|
| **출처 표기** | 원작자 **DeusData** 명시, MIT 라이선스 언급, 설명란에 깃허브 링크 |
| **사칭 금지** | "제가 만들었어요" 절대 금지. "제가 분석/활용해봤어요"로 |
| **수치 검증** | README 수치는 **프로젝트 주장**임을 명시 + 본인 환경 실측치 병기. 신뢰도의 핵심 |
| **오탐 안내** | Windows Defender 오탐(`Trojan:Script/Wacatac.B!ml`) 미리 설명. 62개 중 1개 ML 오탐이며 `gh`, llama.cpp, Godot, MS Go 툴체인도 같은 계열에 걸린다 |
| **보안 고지** | "코드베이스를 읽고 에이전트 설정 파일을 수정한다"는 설계를 솔직히 안내 |
| **코드 노출 주의** | 회사/고객 코드 노출 금지. 공개 오픈소스 레포로 데모 |
| **프라이버시 강조** | "100% 로컬, 코드가 나가지 않음" — 시청자 최대 관심사 |

### 수익 연결 구조
```
유튜브 (무료 유입)
  └→ 깃허브 레포 (EP10/11 완성 코드 = 포트폴리오)
       ├→ 유료 강의 (인프런/클래스101) "MCP 서버 직접 만들기"
       ├→ 전자책 / 노션 템플릿
       ├→ 기업 세미나 / 사내 교육
       └→ 컨설팅 / 외주 유입
```

### 쇼츠/릴스 아이디어
1. 3D 그래프 회전 15초 + "내 코드베이스입니다"
2. 토큰 412,000 → 3,400 카운트다운
3. "리눅스 커널 인덱싱 3분" 타임랩스
4. "안 쓰는 함수 37개 발견" 결과 화면
5. "API 키 0개로 시맨틱 코드 검색"

---

## 12. 수익화 아이디어 (상세)

### 전제 1 — 라이선스

```
MIT License / Copyright (c) 2025 DeusData
✅ 상업적 사용, 수정, 배포, 사내 이용, 서브라이선스 모두 가능
📌 조건: 저작권 표시 + 라이선스 사본 유지
🚫 무보증 (AS-IS)
```
추가 확인사항: `THIRD_PARTY.md`(11KB)의 벤더링 서드파티 라이선스도 함께 확인. **사칭 절대 금지.**

### 전제 2 — 현실 인식

**엔진 자체를 파는 것은 사실상 불가능하다.** MIT로 누구나 무료 입수 가능하고,
원작자가 이미 npm/PyPI/Homebrew 등 8개 채널에 무료 배포 중이며,
21만 줄 C 코드와 기능 경쟁은 승산이 없다.
**수익은 엔진의 "위/옆/주변"에서 나온다.** (오픈소스 수익화의 정석)

---

### 티어 1 — 즉시 시작 가능 (초기자본 ~0, 1~3개월)

#### 아이디어 1. 교육 콘텐츠 ★★★

| 구분 | 내용 | 예상 수익 |
|---|---|---|
| 유튜브 | 11장의 12편 시리즈 | 광고 + 브랜드 협찬 |
| 유료 강의 | 인프런/클래스101/패스트캠퍼스 "MCP 서버 직접 만들기" | 3~8만원 × N명 |
| 전자책 | 리디/부크크/Gumroad "AI 에이전트를 위한 코드 지식 그래프" | 1~3만원 |
| 유료 뉴스레터 | MCP 생태계 주간 큐레이션 | 월 5천~1만원 구독 |
| 기업 세미나 | 사내 교육 2~4시간 | 회당 100~500만원 |

1순위 이유: 초기비용 ~0, 재고 없음, 한 번 만들면 계속 팔림, **다른 모든 수익화의 마케팅 채널**,
한국어 MCP 심화 콘텐츠 공백 선점 가능.

```
1개월  유튜브 EP1~3 (무료, 신뢰 구축)
2개월  EP4~8 + 깃허브에 미니 MCP 서버 공개
3개월  EP10(TS 핸즈온) → 유료 강의 런치
4개월  전자책 + 기업 세미나 영업
```

#### 아이디어 2. 기술 컨설팅 / 도입 서비스 ★★★ (단가 최고)

타겟: AI 코딩 도구 도입을 검토하는 중소 개발팀, 레거시 보유 기업

| 서비스 | 내용 | 가격대 |
|---|---|---|
| 레거시 진단 | 인덱싱 → 아키텍처 리포트 → 기술부채 랭킹 → 리팩토링 로드맵 | 300~1,500만원 |
| 도입 세팅 | 팀 환경 구축 + 공유 아티팩트 CI 파이프라인 + 교육 | 200~800만원 |
| AI 에이전트 워크플로 설계 | Scout/Verify/Auditor 커스터마이징 + 프롬프트/훅 설계 | 500~2,000만원 |
| 월 리테이너 | 지속 모니터링 + 월간 리포트 + 질의응답 | 월 100~400만원 |

**핵심:** 기업은 도구가 아니라 **결과**에 돈을 쓴다.
"무료 도구 설치해드립니다"(X) → "6개월 걸릴 레거시 파악을 2주로 줄이고 리팩토링 리스크 맵을 드립니다"(O)

차별화 무기: 실측 ROI 데이터("토큰 비용 월 N만원 절감"), **3D 그래프 = 경영진 설득용 비주얼**,
`detect_changes` 리스크 맵 = 보고서 하이라이트.

#### 아이디어 3. 오픈소스 선점 → 평판 → 기회 ★★

```
① 한국어 완전 번역 + 한국 개발환경 가이드 (라라벨/스프링/Next.js)
② "awesome-mcp-korea" 큐레이션 레포
③ 미니 MCP 서버(TS) 오픈소스 공개 (EP10 결과물)
④ CBM 래퍼/플러그인 (VSCode 확장, GitHub Action)
⑤ 원본 프로젝트에 PR 기여 → 컨트리뷰터 이력
```
직접 수익은 작지만 깃허브 스타 = 포트폴리오 = 강의/컨설팅/채용 인바운드. 레버리지가 크다.

---

### 티어 2 — 제품화 (3~9개월, 중간 투자)

#### 아이디어 4. "코드베이스 헬스 대시보드" SaaS ★★

Plan C 구조로 React 레이어를 올린다.

```
· 기술부채 점수 추이 (주간/월간)
· 핫스팟 랭킹 (자주 바뀌고 복잡한 파일)
· 데드코드 / 복붙코드 리포트
· PR별 블래스트 반경 자동 코멘트
· 아키텍처 변화 추이 (compare_graphs)
· 슬랙/팀즈 알림 (고위험 변경)
· 멀티 레포 / 조직 뷰 / 권한 관리
· 3D 시각화 (경영진 보고용)
```

| 플랜 | 가격 | 대상 |
|---|---|---|
| Free | 공개 레포 1개 | 유입 |
| Pro | $19/user/월 | 소규모 팀 |
| Team | $49/user/월 | 중견팀 (SSO, 슬랙) |
| Enterprise | 협의 (온프레미스) | 대기업 |

**승부처:** CBM은 CLI/MCP 도구다. **팀 단위 지속 모니터링 + 추이 + 알림 + 권한**은 아예 없다.

```
Frontend  Next.js 15 + React 19 + Tailwind + react-force-graph-3d
Backend   Node/Hono 또는 NestJS → CBM CLI 호출 (--format json)
DB        PostgreSQL (추이) + CBM SQLite (그래프)
Queue     BullMQ (인덱싱 잡)
Deploy    온프레미스 Docker 옵션 필수 (코드 민감성)
```

리스크: 코드 보안 민감 → 온프레미스/셀프호스팅 사실상 필수.
경쟁자(SonarQube, CodeScene, Codacy) 존재 → **"AI 에이전트 최적화"** 포지셔닝으로 차별화. 업스트림 추적 부담.

#### 아이디어 5. PHP/라라벨 전문 분석기 ★★★ (틈새 독점)

Plan B. 한국에 레거시 PHP + 워드프레스가 많은데 **PHP 전문 코드 그래프 분석기는 시장에 거의 없다.**

```
제품명 예: "LaraGraph" / "PHPScope"
핵심 기능: 라라벨 전체 체인 추적, Eloquent 관계 ERD, Blade 변수 흐름,
          워드프레스 훅/필터 추적, PSR-4 매핑, PHP 5.6→8.x 마이그레이션 리포트,
          플러그인 충돌 탐지
수익 모델: PHP 레거시 진단 (건당 200~800만원)
          워드프레스 에이전시 B2B 라이선스 (월 구독)
          라라벨 개발사 대상 SaaS
          해당 분야 강의/전자책
```
경쟁 적음 + 수요 확실.

#### 아이디어 6. 특화 MCP 서버 제품화 ★★

CBM은 범용이라 특정 도메인 깊이가 부족하다.

| 틈새 | 설명 | 수요 |
|---|---|---|
| DB 스키마 그래프 MCP | 테이블↔쿼리↔ORM↔API 추적 | 높음 |
| 프론트엔드 전문 MCP | 컴포넌트 트리, props 흐름, 상태 관리, 디자인 토큰 | 높음 |
| API 스펙 MCP | OpenAPI ↔ 구현 ↔ 테스트 ↔ 클라이언트 SDK 정합성 | 높음 |
| 보안 MCP | 데이터 흐름 기반 취약점 경로 (`DATA_FLOWS` 활용) | 높음 |
| 인프라 전문 MCP | Terraform/K8s/Helm 의존성 + 비용 추정 | 중간 |
| 테스트 커버리지 MCP | 미테스트 고위험 코드 식별 | 중간 |

판매: MCP 레지스트리/Glama 등록(무료 티어) → Pro 기능 유료 / 팀 라이선스 / 맞춤 개발

---

### 티어 3 — 큰 그림 (9개월+, 고위험·고수익)

**아이디어 7. AI 코드 리뷰 봇 ★**
PR 열림 → 블래스트 반경 계산 → 영향 심볼 코드만 LLM에 전달(토큰 절약) → 구조 인식 리뷰 코멘트.
$10~30/repo/월. 경쟁 강력(CodeRabbit, Greptile). 차별화 = 그래프 기반 구조 인식 + 토큰 효율 가격경쟁력.

**아이디어 8. 레거시 마이그레이션 자동화 플랫폼 ★**
PHP 5→8, jQuery→React, 모노리스→MSA. 의존성 분석 → 마이그레이션 순서 자동 계획 → 단계별 실행.
프로젝트당 수천만~수억. 난이도 매우 높음.

**아이디어 9. 교육 플랫폼 "오픈소스 탐험" ★**
유명 오픈소스를 3D 그래프로 탐험하며 학습. 게임화 + 퀘스트 + 수료증.
월 $15~30 구독. 대상: 주니어 개발자, 부트캠프 B2B. 차별화 = 3D 비주얼 학습 경험.

---

### 종합 비교

| # | 아이디어 | 난이도 | 초기비용 | 수익 속도 | 최대 규모 | 추천 |
|---|---|---|---|---|---|---|
| 1 | 교육 콘텐츠 | 낮음 | 거의 0 | 빠름 | 중 | ★★★ |
| 2 | 컨설팅/도입 | 중간 | 거의 0 | 빠름 | 중 | ★★★ |
| 5 | PHP 전문 분석기 | 중간 | 낮음 | 보통 | 중상 | ★★★ |
| 3 | 오픈소스 선점 | 낮음 | 거의 0 | 느림 | 간접 | ★★ |
| 4 | 헬스 대시보드 SaaS | 높음 | 중 | 느림 | 큼 | ★★ |
| 6 | 특화 MCP 서버 | 중간 | 낮음 | 보통 | 중상 | ★★ |
| 7 | AI 코드 리뷰 봇 | 높음 | 중 | 느림 | 큼 | ★ |
| 8 | 마이그레이션 플랫폼 | 매우 높음 | 큼 | 매우 느림 | 매우 큼 | ★ |
| 9 | 교육 플랫폼 | 높음 | 중 | 느림 | 중상 | ★ |

### 추천 루트

```
1~3개월    유튜브 + 깃허브 (무료, 신뢰/평판 구축) — 실측 데이터 축적, 미니 MCP 서버 공개
3~6개월    유료 강의 런치 + 컨설팅 영업 개시 — 현금흐름 확보 + 시장 니즈 직접 청취
6~12개월   PHP 전문 분석기 또는 특화 MCP 서버 제품화 — 컨설팅 고객이 첫 고객
12개월+    SaaS 확장 또는 대형 프로젝트
```

**핵심 원칙 4가지**
1. **콘텐츠 먼저, 제품 나중** — 평판이 모든 영업의 기반
2. **컨설팅으로 시장 검증** — 돈을 받으면서 고객의 진짜 니즈를 배운다
3. **온프레미스 옵션 필수** — 코드는 기업의 가장 민감한 자산
4. **출처·라이선스 정직하게** — 원작자 존중이 장기 브랜드를 지킨다

### 리스크 및 대응

| 리스크 | 대응 |
|---|---|
| 원작자의 상용화 가능성 | 엔진에 의존하지 않고 **자체 레이어**에 가치를 둔다 |
| 업스트림 변경으로 깨짐 | CLI JSON 출력에 **어댑터 레이어**를 두고 버전 핀 고정 |
| 코드 보안 우려로 SaaS 거부 | 온프레미스/셀프호스팅 플랜을 처음부터 준비 |
| 경쟁자 진입 | 도메인 특화(PHP/라라벨, 한국 시장)로 참호 구축 |
| 수치 과대 인용 → 신뢰 손상 | **반드시 본인 환경 실측치 병기** (가장 중요) |
| 라이선스 위반 | MIT 고지 유지 + `THIRD_PARTY.md` 확인 + 사칭 금지 |

---

## 13. 부가 기능 메모

### 팀 공유 그래프 아티팩트
`.codebase-memory/graph.db.zst` — zstd 압축 그래프 스냅샷을 레포에 커밋하면 팀원이 재인덱싱을 생략한다.
- opt-in: `index_repository`가 `persistence: true`일 때만 생성 (기본 `false`)
- 포맷: SQLite → 인덱스 제거 → `VACUUM INTO` 압축 → zstd (8~13:1)
- 2단계 티어: **Best**(`zstd -9` + 인덱스 제거 + VACUUM) / **Fast**(`zstd -3`, 워처 증분 갱신용)
- 부트스트랩: 로컬 DB가 없고 아티팩트만 있으면 임포트 후 증분 인덱싱
- 머지 충돌 없음: `.codebase-memory/.gitattributes`에 `merge=ours` 자동 생성
- ⚠️ **커밋 주기 주의**: 매 인덱싱마다 재작성되고 git은 매번 새 blob으로 저장한다.
  한 팀은 이 경로 ~350커밋에서 ~6GB에 도달했다. 릴리스/마일스톤/야간 잡 단위 주기를 정할 것
- Git LFS 사용 시 **레포 루트** `.gitattributes`에서 추적하고 `.codebase-memory/.gitattributes`는 그대로 둔다
  (가까운 파일이 `merge=ours`를 계속 공급, 루트에서는 `filter`만 가져온다).
  `.zst`만 추적하고 `artifact.json`은 제외. GitHub LFS는 저장용량/대역폭 과금 + 객체 prune 불가 + 팀원 전원 `git lfs install` 필요
- 원하지 않으면 `.codebase-memory/`를 `.gitignore`에 추가

### 세션 조정 데몬
Claude Code, Codex, OpenCode 등 모든 클라이언트가 **계정당 데몬 1개**를 공유한다.
opt-in 설정 없음. 첫 세션이 시작하고 마지막 세션이 종료한다.
데몬이 워처·공유 인덱싱 잡·UI를 소유한다.

로그 (`${CBM_CACHE_DIR}/logs`, 기본 `~/.cache/codebase-memory-mcp/logs`)

| 파일 | 내용 |
|---|---|
| `cbm-daemon.log` | 데몬 라이프사이클, 워처/인덱싱, UI, 리소스, 에러 |
| `daemon-conflicts.ndjson` | 정확한 빌드 / 조정 ABI / 캐시 루트 충돌 |
| `activation-events.ndjson` | install/update/uninstall 활성화 진행·결과 |

모든 활성 CBM 프로세스는 **동일 버전·빌드·조정 ABI·canonical 캐시 루트**여야 한다.
`install`/`update`/`uninstall`만 예외로, 다운로드·검증·스테이징을 먼저 하고
활성화 단계에서 모든 프로세스 종료를 유한 데드라인까지 대기한 뒤 배타적으로 교체한다.

### 인프라 코드 인덱싱
Dockerfile, Kubernetes 매니페스트, Kustomize 오버레이를 그래프 노드로 인덱싱.
K8s kind는 `Resource` 노드, Kustomize 오버레이는 `Module` 노드 + 참조 리소스로 `IMPORTS` 엣지.

### 크로스 레포 인텔리전스
동일 스토어에 인덱싱된 여러 레포를 `CROSS_*` 엣지로 연결.
멀티 갤럭시 3D UI 레이아웃, 플릿 전체 아키텍처 요약(서비스/라우트/의존성).

### 패키지/모듈 해석
`@myorg/pkg`, `github.com/foo/bar`, `use my_crate::foo` 같은 bare specifier를 매니페스트 스캔으로 해석:
`package.json`, `go.mod`, `Cargo.toml`, `pyproject.toml`, `composer.json`, `pubspec.yaml`,
`pom.xml`, `build.gradle`, `mix.exs`, `*.gemspec`

### 릴리스 보안 파이프라인
- **VirusTotal** — 8개 릴리스 제품 × 3후보(unstripped/debug-stripped/stripped) = 24개 실행파일을
  smoke/soak 전에 스캔. 선택된 실행파일은 **SHA-256 변경 없이** 패키징.
  컨테이너에서 추출된 모든 객체(`install.sh`, `install.ps1`, `LICENSE`, `THIRD_PARTY_NOTICES.md`,
  MCPB `manifest.json`, UI 애셋)도 스캔
- **SLSA Level 3** — 빌드 provenance. 검증:
  `gh attestation verify <file> --repo DeusData/codebase-memory-mcp --signer-workflow DeusData/codebase-memory-mcp/.github/workflows/_build.yml`
- **Sigstore cosign** — 키리스 서명, 번들 동봉
- **SHA-256 checksums** — `checksums.txt`, 두 install 스크립트가 추출 전 검증
- **CodeQL SAST** — 미해결 알림이 있으면 릴리스 파이프라인 차단
- 언어 런타임 의존성 체인 없음 — 라이브러리는 컴파일 타임 벤더링

---

## 14. 솔직한 평가 / 주의사항

| 항목 | 내용 |
|---|---|
| **README 수치는 자체 주장** | 테스트 8,050개, 99.2% 토큰 절감, 162개 언어 등은 문서 기준이며 직접 검증하지 않았다. 실측은 `docs/MEASURING_SAVINGS.md` 참고 |
| **Windows Defender 오탐** | `Trojan:Script/Wacatac.B!ml`. 프로젝트는 ~62개 중 61개 clean인 알려진 ML 오탐으로 설명하고 VirusTotal 링크로 증빙한다. `gh`, llama.cpp, Godot, MS Go 툴체인도 같은 계열에 걸린다 |
| **설계상 권한 범위** | "코드베이스를 읽고 에이전트 설정 파일에 쓴다"는 것을 README가 직접 밝힌다. 45개 클라이언트 config를 자동 수정하는 것은 강력하지만 신뢰가 필요한 동작. MIT 오픈소스이고 전체 소스가 공개되어 감사 가능 |
| **업스트림 추적 필요** | 이 레포는 `DeusData`의 복사본이므로 업스트림 업데이트는 수동으로 따라가야 한다 |
| **확장 난이도** | 순수 C 21만 줄. 기능 추가는 C 작업을 요구한다 |
| **구조만 안다** | 비즈니스 로직의 정합성이나 설계 의도는 판단하지 못한다. 그것은 LLM의 몫 |
| **인덱스 신선도** | 에이전트가 오래된 그래프를 신뢰하면 잘못된 결론에 이른다. `index_status`/`check_index_coverage` 확인 습관 필요 |

---

## 15. 결론

> **이것은 "AI 에이전트용 코드베이스 두뇌 이식 키트"다.**
> grep으로 더듬던 에이전트를 **구조를 아는 에이전트**로 바꿔주는 인프라 레이어.

활용 가치 3축:
1. **실사용 생산성 도구** — 토큰 절감, 레거시 파악, 리팩토링 안전망, MSA 추적
2. **학습 교재** — C 21만 줄로 구현된 그래프DB + Cypher 파서 + MCP 서버 + LSP + 임베딩 검색,
   그리고 SLSA 3 / Sigstore / CodeQL / OpenSSF를 갖춘 프로덕션급 OSS 운영 모범 사례
3. **콘텐츠·비즈니스 소재** — 유튜브/강의/컨설팅/제품화. 특히 교육 콘텐츠 → 컨설팅 → 제품화 순서 추천

---

*문서 작성: Claude Code 세션 분석 결과 정리*
*원작 프로젝트: https://github.com/DeusData/codebase-memory-mcp (MIT License, Copyright (c) 2025 DeusData)*
*이 레포: https://github.com/bmshin94/codebase-memory-mcp*
