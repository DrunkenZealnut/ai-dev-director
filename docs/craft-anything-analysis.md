# Craft Anything 프로젝트 분석

- 원본: https://github.com/storytold/craft-anything (커밋 `0003362`, 2026-11 기준 `main`)
- 복사 위치: 이 레포의 [`craft-anything/`](../craft-anything/) 폴더 (원본의 git 추적 파일 870개를 그대로 복사, `.git` 히스토리 제외)
- 라이선스: MIT OR Apache-2.0 (원본 `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE`, `ATTRIBUTION.md`를 함께 보존)
- 버전: 워크스페이스 `0.4.0`, 상태 **early alpha**

## 1. 한 줄 요약

**"어떤 웹사이트·앱·게임이든 어떻게 동작하는지 분석하고, 코딩 에이전트로 나만의 버전을 다시 만드는"** 순수 Rust
리버스 엔지니어링 + 리빌드 엔진입니다. 데스크톱 앱(egui), CLI, MCP 서버, WebAssembly 뷰어가 모두 **하나의 엔진**과
**하나의 도구 레지스트리**를 공유합니다.

## 2. 핵심 워크플로

```
Study  ──▶  Build  ──▶  Check  ──▶  Ready (Refine 반복)
원본 분석     코딩 에이전트가     원본과 비교       결과 실행·미리보기,
(페이지/코드/  새 프로젝트 생성   (3가지 화면 너비,  추가 요청 시 다시 다듬기
 파일/실행)   (Claude Code 우선) 게임 테스트 등)
```

- **Clean room 모드(기본값)**: 원본 내부(코드·파일·페이지 소스)를 절대 읽지 않고, 스크린샷·보이는 텍스트·레이아웃 등
  "방문자가 볼 수 있는 것"만 사용합니다. 허용 도구 목록은 `crates/engine/src/clean_room.rs`에 고정돼 있습니다.
- **Deep 모드**: 개인 용도로 모든 도구 사용. 자체 디스어셈블러/디컴파일러, 선택적으로 Ghidra, JS 번들 분석 등.
- 빌더: 설치된 `claude` CLI를 그대로 실행(사용자의 기존 로그인 사용). Codex 어댑터는 있으나 실사용 검증 전.
  분석/질문용 LLM은 Anthropic, OpenAI, OpenRouter, 로컬 OpenAI 호환 서버를 지원합니다.

## 3. 규모

| 항목 | 값 |
|---|---|
| Rust 코드 | 약 **278,600줄** |
| 추적 파일 | 870개 (~22MB, 이미지/아이콘 포함) |
| `#[test]` 수 | 약 2,900개 (STATUS.md 기준 감사 시점 923개 통과 후 계속 증가) |
| 도구 수 | README 기준 320개 (엔진 300 + AI/setup 레이어) |
| 지원 OS | macOS · Windows · Linux · FreeBSD · Web(wasm) |
| 툴체인 | Rust 1.97, edition 2024, resolver 3 |

### 크레이트별 코드량

| 크레이트 | 레이어 | 줄 수 | 역할 |
|---|---|---:|---|
| `evidence` | L0 | 907 | 근거(Evidence) 레코드, 정규 ID, 신뢰도, 미확인 항목, `ToolError` 에러 계약 |
| `formats` | L0 | 52,474 | 의존성·I/O 없는 순수 파서: ELF/PE/Mach-O/WASM/DEX/APK/ASAR/plist/.NET/Go/SQLite 등 |
| `disasm` | L1 | 5,845 | 자체 네이티브 엔진: 디코딩, 함수, CFG, xref, 콜 그래프, 의사 C |
| `js` | L1 | 5,612 | JS/Electron/웹 번들 분석 (oxc 기반 파싱, 모듈 그래프, IPC, 소스맵, 언번들링) |
| `analysis` | L2 | 10,556 | 파싱 결과 → 근거 생성 분석기 (트리아지, 비교, 앱 그래프, 리빌드 원장) |
| `platform` | L2 | 7,031 | 프로세스 소유/PTY 캡처, 샌드박스, 엔진 저장소, 원자적 파일 쓰기 |
| `engines` | L3 | 9,500 | 외부 엔진 어댑터: Ghidra, adb, Chromium(CDP), Frida, JADX, Apktool… |
| `engine` | L4 | 58,303 | 세션, 프로젝트(`.cany`), 워크스페이스, inbox(질문/승인), **ToolSpec 레지스트리**, 설정, 동의 |
| `ai` | L5 | 32,502 | AI 백엔드, 에이전트 루프, crew, **리빌드 워크플로** (`rebuild/`: study/check/stacks…) |
| `mcp` | L5 | 1,528 | MCP 서버 (progressive toolset, resources, prompts) |
| `setup` | L5 | 2,749 | 코딩 에이전트 앱 연동: 감지·계획·병합·백업·롤백·doctor |
| `ui-egui` | L6 | 22,195 | egui UI (Simple/Advanced). egui/eframe/winit/rfd를 쓸 수 있는 유일한 크레이트 |
| `testkit` | – | 8,364 | 테스트 헬퍼, 합성 샘플 빌더 |
| `apps/craft-anything` | – | 1,088 | 데스크톱 앱 진입점 |
| `apps/craft-anything-cli` | – | 2,878 | CLI + `mcp` 서브커맨드 (`rebuild_cmd`, `ai_cmd`, `setup_cmd`) |
| `apps/craft-anything-web` | – | 435 | 브라우저 뷰어 (Trunk, wasm) |
| `xtask` | – | 1,233 | `cargo xtask layers / tools / wasm / secrets / ci / version / pending` |

## 4. 아키텍처 특징

1. **엄격한 레이어링 (L0 → L6)**: 하위 레이어는 상위 레이어를 참조할 수 없고, `cargo xtask layers`로 CI에서 강제합니다.
   L0–L2는 wasm 빌드가 깨지면 안 됩니다(`cargo xtask wasm`).
2. **"Everything is a tool"**: 사용자에게 보이는 모든 동작은 `crates/engine/src/tools/<area>.rs`의 `ToolSpec`
   (id, 제목, 요약, 초보자용 설명, toolset, effects, requires, schemars 파라미터 구조체, `run`)입니다.
   UI·CLI·MCP·AI 에이전트가 모두 **id로 디스패치**하므로, 도구 하나를 추가하면 네 표면에 동시에 노출됩니다.
3. **Evidence가 결과물**: 분석 도구는 `ToolOutput::Evidence`(신뢰도, 한계, 커버리지, 위치)를 반환합니다.
   큰 결과는 요약 우선 + `offset/limit` 페이징.
4. **정직한 Effects와 동의**: `EXECUTES_FOREIGN_CODE`, `MUTATES_DEVICE` 같은 부작용을 선언하면 사용자 동의를 거칩니다.
5. **Never crash**: 적대적 입력이 주 입력이므로 프로덕션 코드에서 `unwrap/expect/panic!/인덱싱`을 금지(크레이트 루트 deny),
   checked 연산·재귀 깊이 제한·할당 상한을 원칙으로 합니다. 크래시 수정마다 회귀 테스트 필수.
6. **확장 포인트는 trait + registry**: 타깃 종류, 체크, 빌더, 에셋 공급자, 외부 엔진을 enum `match`가 아니라
   trait 구현 + 등록 한 줄로 추가. 카탈로그(스택, 게임 엔진 레시피, 모델 가격 등)는 `stacks.toml`, `toolchains.toml`처럼
   `include_str!`로 임베드한 데이터로 관리합니다.
7. **Headless first**: GUI에서 가능한 모든 흐름(리빌드, 질문 응답, 엔진 설치, 결과 실행)은 `Engine::call`과 CLI로도 가능.

## 5. 품질/운영 체계

- **CI**: `.github/workflows/`에 `ci.yml`, `freebsd.yml`, `windows-arm64.yml`, `packaging-lint.yml`, `release.yml`,
  `sonarcloud.yml`. 공통 게이트는 Crafting Apps 공용 워크플로(`@main`)를 따릅니다.
- **시크릿 스캔**: `cargo xtask secrets [--history]` + `.githooks` pre-commit 훅.
- **패키징**: `packaging/` 아래 Windows / macOS(서명·공증) / Linux / FreeBSD / Web.
- **vendor 패치**: `vendor/wgpu-hal` — Metal 셰이더 버전 문제 우회용 임시 패치(upstream wgpu #10687 대기).
- **문서 체계**: `docs/ROADMAP.md`(R<마일스톤>.<n> id, 담당자, claim 규칙), `docs/STATUS.md`(무엇이 어떻게 검증됐는지),
  `docs/tool-status.md`, `docs/extending.md`, `docs/pitfalls.md`. 9개 언어 README.
- **에이전트 친화적 레포**: `AGENTS.md`/`CLAUDE.md`에 필수 규칙을 명시하고, AI 에이전트가 ROADMAP 항목을 claim해서
  작업하는 흐름을 전제로 합니다.

## 6. 현재 상태 (README/STATUS 기준)

- **동작함**: 데스크톱(Simple/Advanced), CLI, MCP 서버, 웹사이트·앱·게임 Study→Build→Check→Refine (Claude Code 빌더),
  3가지 화면 너비 웹 비교, Unity/Unreal/Godot 게임 리빌드, Ghidra 연동 Deep 모드, fal.ai 이미지/3D 생성(옵트인),
  플러그인(MCP 서버, `wasm-plugins` 옵션의 WASM 컴포넌트).
- **부분적**: API 키/다른 코딩 에이전트 빌드(구현됐으나 실사용 미검증), 앱·게임 동작 비교, 일부 엔진 도구는 스탠드인으로만 테스트.
- **다음**: managed tier.

## 7. ai-dev-director 관점에서의 시사점

ai-dev-director(Python, 멀티 에이전트 디렉터 + 대시보드)와 비교했을 때 참고할 만한 점:

| 주제 | craft-anything 방식 | ai-dev-director에 적용 아이디어 |
|---|---|---|
| 단일 디스패치 | 모든 기능 = `ToolSpec`, UI/CLI/MCP/에이전트가 id로 호출 | 대시보드·CLI·에이전트 액션을 하나의 레지스트리로 통합 |
| 근거 기반 출력 | Evidence(신뢰도·한계·위치) | 에이전트 결과에 근거/신뢰도 메타데이터 부여 |
| 승인 inbox | 질문/승인 카드, 처음 보는 명령은 묻기 | 위험 명령에 대한 승인 큐 |
| 레이어 강제 | `cargo xtask layers` | import 규칙 린트(예: import-linter) |
| 로드맵 claim | ROADMAP id + claim + STATUS 검증 기록 | 에이전트 작업 배정/검증 로그 형식으로 차용 |
| 시크릿 스캔 | xtask + git hook | pre-commit 시크릿 스캔 추가 |

## 8. 참고 / 주의

- 이 사본은 **분석용 스냅샷**입니다. 원본과 동기화되지 않으며, `craft-anything/` 안의 코드는 수정하지 않았습니다.
- 이번 분석에서는 빌드/테스트를 실행하지 않았습니다(Rust 1.97 툴체인 및 GUI 의존성 필요). 수치는 소스 정적 집계와
  원본 문서(README, AGENTS.md, docs/STATUS.md, docs/ROADMAP.md)에 근거합니다.
- 원본 README의 "Responsible use": 소유하거나 분석 권한이 있는 소프트웨어에만 사용하고, DRM/로그인/안티치트 우회는 하지 않습니다.
