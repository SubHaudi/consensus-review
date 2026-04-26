# consensus-review

> **Multi-agent document review skill** — N명의 독립 LLM 리뷰어를 병렬로 돌려 문서 결함을 탐지하고, 합의 수준으로 우선순위를 매깁니다.

> "단일 LLM 리뷰는 매번 다른 결함을 놓친다. 여러 명이 동시에 보면 놓치는 게 줄어든다."
>
> "다수결은 함정이다 — 한 명만 본 결함도 실제 결함일 수 있다."
>
> "합의 수준(consensus) 자체가 무료 품질 신호다."

Kiro, Claude Code, Cursor, Codex CLI, Gemini CLI, OpenCode 등 Open Agent Skills 호환 도구 어디서든 바로 사용할 수 있습니다.

---

## Why consensus-review?

문서 검토에서 가장 흔한 실패는 "한 명의 LLM에게 잘 리뷰해달라고 비는 것"입니다. 6개 LLM × 3개 벤치마크(법률, 사실오류, 번역) × 10,200+ 정답 결함으로 검증한 결과, 같은 모델을 같은 프롬프트로 5번 돌리면 **매번 다른 결함**을 놓쳤습니다. 놓침은 "모르는 것"이 아니라 "확률적으로 놓치는 것"이었습니다.

consensus-review는 이 관찰을 바탕으로 설계되었습니다:

- 📈 **재현율 +6.5 ~ +30.6%p 향상** (단일 리뷰 대비, 18/18 조합에서 통계적으로 유의)
- 🎯 **합의 수준으로 정밀도 신호 자동 부여** — 🔴 (전원 합의) → 🟡 (다수) → ⚪ (소수) 순 정밀도 단조 감소
- ⚖️ **다수결 함정 회피** — 합집합(Union) 기반이라 한 명만 본 결함도 ⚪로 살려둠
- 🧩 **도구 중립** — Open Agent Skills 표준, 어느 에이전틱 도구에서도 작동

상세 근거: [`references/benchmark_summary.md`](references/benchmark_summary.md)

---

## What's included

```
consensus-review/
├── SKILL.md                          # 스킬 진입점 (에이전트가 읽음)
├── README.md                         # 이 파일 (사람이 읽음)
├── LICENSE                           # MIT
├── install.sh                        # 도구별 원샷 설치 스크립트
├── prompts/
│   ├── review.md                     # 서브에이전트용 리뷰 프롬프트
│   └── aggregate.md                  # 메인 에이전트용 집계 프롬프트
├── references/
│   └── benchmark_summary.md          # 설계 근거 (실험 결과)
└── examples/
    ├── output_template.md            # 사용자 보고 템플릿
    ├── sample_aws_architecture.md    # 샘플 입력 문서 (AWS 아키텍처)
    └── sample_run.md                 # 샘플 실행 결과 (데모)
```

---

## Installation

### One-line install (recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/SubHaudi/consensus-review/main/install.sh | bash -s <tool>
```

Replace `<tool>` with one of the supported tools below.

| Tool | Argument | Install path |
|---|---|---|
| Kiro (global) | `kiro` | `~/.kiro/skills/consensus-review` |
| Kiro (workspace) | `kiro-local` | `./.kiro/skills/consensus-review` |
| Claude Code (global) | `claude-code` | `~/.claude/skills/consensus-review` |
| Claude Code (project) | `claude-local` | `./.claude/skills/consensus-review` |
| Cursor | `cursor` | `./.cursor/skills/consensus-review` |
| Codex CLI (user) | `codex` | `~/.agents/skills/consensus-review` |
| Codex CLI (project) | `codex-local` | `./.agents/skills/consensus-review` |
| Gemini CLI | `gemini` | `./.gemini/skills/consensus-review` |
| OpenCode | `opencode` | `./.opencode/skills/consensus-review` |
| GitHub Copilot | `copilot` | `./.github/skills/consensus-review` |

**Example**:

```bash
# Install globally for Kiro
curl -fsSL https://raw.githubusercontent.com/SubHaudi/consensus-review/main/install.sh | bash -s kiro

# Install for Claude Code in current project
curl -fsSL https://raw.githubusercontent.com/SubHaudi/consensus-review/main/install.sh | bash -s claude-local
```

Run without arguments to see all options: `bash install.sh --help`.

### Manual installation

If you prefer not to pipe curl to bash, clone the repo and copy manually.

**Kiro**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p ~/.kiro/skills
cp -r consensus-review ~/.kiro/skills/
```

**Claude Code**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p ~/.claude/skills
cp -r consensus-review ~/.claude/skills/
```

**Cursor**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p .cursor/skills
cp -r consensus-review .cursor/skills/
```

> **Note:** Cursor skills require setup:
> 1. Switch to Nightly channel in Cursor Settings → Beta
> 2. Enable Agent Skills in Cursor Settings → Rules

**Codex CLI**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p ~/.agents/skills
cp -r consensus-review ~/.agents/skills/
```

**Gemini CLI**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p .gemini/skills
cp -r consensus-review .gemini/skills/
```

> **Note:** Gemini CLI skills require preview version:
> ```bash
> npm i -g @google/gemini-cli@preview
> ```
> Then run `/settings` and enable "Skills".

**OpenCode**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p .opencode/skills
cp -r consensus-review .opencode/skills/
```

**GitHub Copilot**
```bash
git clone https://github.com/SubHaudi/consensus-review.git
mkdir -p .github/skills
cp -r consensus-review .github/skills/
```

### Other Open Agent Skills compatible tools

해당 도구의 스킬 디렉토리 (`skills/` 또는 도구별 표준 경로)에 `consensus-review` 디렉토리 전체를 복사하세요.

---

## Quick Start

설치 후 5단계로 시작할 수 있습니다.

### 1. 스킬이 인식되는지 확인

에이전트를 실행하고 스킬 목록에 `consensus-review`가 있는지 확인합니다 (도구별로 방법 다름 — Kiro는 `AGENT STEERING & SKILLS` 패널, Claude Code는 `/skills list`).

### 2. 문서 준비

리뷰 받고 싶은 문서를 텍스트 파일로 준비합니다. 크기 제한은 없지만, 모델 컨텍스트 한계를 고려하세요 (대부분 60K 단어 이내 권장).

지원 문서 유형:

- 사업계획서 / business plan
- 기능정의서 / feature spec / PRD
- 구현계획서 / implementation plan / technical design doc
- 계약서 / NDA / legal document
- AWS 아키텍처 문서 / architecture spec
- 일반 문서 (5가지 결함 카테고리로 범용 검토)

### 3. 에이전트에게 요청

```
이 사업계획서를 리뷰해줘.
[문서 첨부 또는 경로]
```

또는 영어로:

```
Please review this AWS architecture spec for missing requirements,
inconsistencies, and unclear language.
[document]
```

스킬이 자동으로 트리거됩니다.

### 4. 결과 확인

에이전트가 다음 순서로 처리합니다:

1. 문서 언어(ko/en)와 유형 감지
2. 서브에이전트 3명을 **병렬**로 실행 — 동일 프롬프트, 독립 추론
3. 3명의 결과를 합의 Tier(🔴/🟡/⚪)로 집계
4. Markdown 리포트 출력

샘플 출력:

```markdown
# 📋 Consensus Review 결과
총 11개 고유 이슈 (🔴 4, 🟡 5, ⚪ 2)

## 🔴 High Confidence — 전원 합의 (4개)
### ISS-1: 가용영역(AZ) 간 라우팅 조건 누락
...

## 🟡 Needs Review — 다수 합의 (5개)
...

## ⚪ Low — 소수 지적 (2개)
...

💡 🔴부터 우선 검토하세요.
```

전체 예시: [`examples/sample_run.md`](examples/sample_run.md)

### 5. 설정 바꾸기 (선택)

에이전트에게 채팅으로 요청:

- `"리뷰어 5명으로 해줘"` → N=5 (기본 N=3)
- `"보안 이슈만 봐줘"` → 프롬프트에 지시 추가
- `"한국어로 출력해줘"` → 최종 리포트를 한국어로

---

## Configuration

`SKILL.md`의 기본값을 수정해 팀/조직에 맞출 수 있습니다.

| 설정 | 기본값 | 변경 가능 위치 |
|---|---|---|
| 리뷰어 수 N | 3 | SKILL.md Step 2, 또는 사용자 요청으로 런타임 변경 |
| 모델 | 사용자 설정 따라감 | 에이전트 툴 설정 |
| 출력 언어 | 문서 언어 감지 (ko/en) | 사용자 요청으로 강제 가능 |
| Tier 기준 | 🔴=3/3, 🟡=2/3, ⚪=1/3 | `prompts/aggregate.md` 변경 시 |
| 결과 파일 저장 | `./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md` | SKILL.md Step 4 |

### 리뷰 결과 파일 — .gitignore 팁

스킬은 리뷰할 때마다 문서와 같은 디렉토리에 `consensus-review-*.md` 파일을 남깁니다. 버전 관리에서 제외하고 싶다면 `.gitignore`에 다음 한 줄 추가:

```gitignore
consensus-review-*-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]-[0-9][0-9][0-9][0-9][0-9][0-9].md
```

리뷰 이력을 같이 커밋하고 싶다면 이 줄은 넣지 마세요. 타임스탬프로 파일이 겹치지 않으므로 누적 보관에도 문제 없습니다.

---

## How it works

이 스킬의 설계는 [`references/benchmark_summary.md`](references/benchmark_summary.md)에 정리된 실험 결과를 기반으로 합니다.

핵심 결정 3가지:

1. **합집합(Union) 사용, 다수결 금지** — 다수결은 재현율을 최대 29%p 깎습니다. 1명만 본 결함도 ⚪로 살려둡니다.
2. **N=3 기본값** — N=5 대비 재현율의 93~97%를 60% 비용으로 달성합니다.
3. **자유 텍스트 ISS-N 포맷** — JSON 스키마 강제 대신. 다단계 파이프라인(리뷰→집계→보고)에서 파싱 실패 리스크를 제거하고 리뷰어 원문 표현을 보존합니다.

실험 결과 요약:

| 측정 | 결과 |
|---|---|
| 단일 대비 재현율 향상 | +6.5 ~ +30.6%p (18/18 조합 유의) |
| 🔴 정밀도 (FAVA 기준) | 80~90% |
| 🟡 정밀도 | 30~40% |
| ⚪ 정밀도 | 15~20% |
| 확률적 실패 검증 | 18/18 조합에서 이항분포 기각, 베타-이항 적합 |

---

## Limitations

- **체계적 실패 영역 복구 불가** — 모든 리뷰어가 똑같이 놓치는 결함(CLAUSE에서 약 10~16%)은 합집합으로도 해결되지 않습니다. 모델 업그레이드 또는 RAG 기반 근거 검색이 필요합니다.
- **CLAUSE 같은 도메인의 낮은 정밀도** — 모델이 잠재적 결함을 공격적으로 지적하는 경향. Tier 분류로 우선순위는 가능하지만 절대 개수를 줄이려면 별도 필터링 필요.
- **한국어 벤치마크 검증 진행 중** — 실험은 영어 기반. 스킬은 한/영 양쪽을 지원하지만 한국어 도메인에서 동일한 패턴이 유지되는지는 후속 검증 항목입니다.

---

## Contributing

이슈와 PR을 환영합니다. 특히 다음 영역의 기여가 유용합니다:

- 한국어 문서 실전 테스트 결과
- 문서 유형별 프롬프트 튜닝 (사업계획서, 기능정의서 등)
- 새로운 에이전틱 도구 호환성 검증 및 설치 명령 추가

## License

MIT License. 자세한 내용은 [LICENSE](LICENSE) 참조.

## References

이 스킬의 설계 근거가 된 실험 (Amazon Bedrock 기반):

- **Models**: Claude Opus 4.6 / Sonnet 4.6 / Haiku 4.5, Kimi K2.5, DeepSeek V3.2, Mistral Large 3
- **Benchmarks**:
  - CLAUSE (Roy Choudhury et al., 2026 EACL Findings) — 법률 결함
  - FAVA (Mishra et al., 2024 NAACL) — 사실 오류
  - WMT MQM (Freitag et al., 2021 TACL) — 번역 품질
- **Background**:
  - Self-consistency (Wang et al., 2023 ICLR) — 선택 과제용 다수결
  - Large Language Monkeys (Brown et al., 2024) — 반복 샘플링 coverage
