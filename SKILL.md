---
name: consensus-review
description: Review Korean/English documents for defects by running N=3 independent LLM reviewers as parallel subagents, then aggregate findings with consensus-tier labels (🔴 high / 🟡 medium / ⚪ low). Use when the user asks to 리뷰해줘, 문서 검토, 놓친 게 있는지 확인, 기능정의서 리뷰, 요구사항 검토, 사업계획서 검토, 구현계획서 검토, review a document, find defects, audit a spec, check for issues in a PRD / RFC / NDA / requirements / architecture / implementation plan. Trigger for filenames like requirements.md, spec.md, prd.md, design.md, architecture.md, or any document review request. The skill increases recall (catches more defects than a single reviewer) and provides tier-based priority for human review.
---

# Consensus Review

여러 명의 독립 리뷰어를 병렬로 돌려 문서 결함을 탐지하고, 합의 수준(🔴/🟡/⚪)으로 우선순위를 매기는 스킬입니다. 단일 LLM이 매번 다르게 결함을 놓치는 **확률적 커버리지(stochastic coverage)** 문제를 해결하기 위해 설계되었습니다.

## 왜 이 스킬을 쓰는가

단일 LLM 리뷰는 매번 다른 결함을 놓칩니다. 같은 문서를 같은 프롬프트로 5번 리뷰시키면 매번 다른 결함 목록이 나옵니다. 이는 "모델이 모르는 것"이 아니라 "매번 다르게 놓치는 것"이기 때문입니다.

실험적으로 검증된 사실 (6개 LLM × 3개 벤치마크 × 10,200+ 결함):

- **독립 N명 리뷰 후 합집합(Union)**: 단일 대비 재현율 +6.5~30.6%p 향상
- **다수결(majority vote)은 오답**: k≥3 합의만 인정하면 재현율이 최대 29%p 감소
- **합의 수준 = 품질 신호**: 🔴 (전원 합의) → 🟡 (다수) → ⚪ (소수) 순으로 정밀도 단조 감소

따라서 이 스킬은 **합집합(Union) + 합의 Tier 분류**를 기본 원칙으로 씁니다. 상세 근거는 `references/benchmark_summary.md`를 참조하세요.

## 언제 이 스킬을 실행하는가

사용자가 다음 같은 요청을 했을 때 이 스킬을 자동으로 실행하세요:

- "이 문서 리뷰해줘", "이 스펙 검토해줘", "놓친 부분 없는지 봐줘"
- "사업계획서 / 기능정의서 / 구현계획서 / PRD / RFC / NDA 검토"
- "이 문서에 결함 있는지 확인", "issues in this doc", "review this spec"
- 긴 문서(1,000 단어 이상)를 붙여넣고 "체크해줘" 류의 요청

단, **단순한 문법 교정, 오탈자 수정, 번역 수정**처럼 "가벼운 편집 요청"에는 쓰지 마세요. 이 스킬은 결함 탐지(defects detection)용이고, 리뷰 한 번에 N배 토큰을 사용하므로 가벼운 요청에는 과합니다.

## 실행 단계

### Step 1. 문서 준비

사용자가 준 문서를 확인하세요. 파일 경로면 읽고, 인라인 텍스트면 그대로 사용합니다. 문서의 **언어(한국어/영어)**와 **유형 힌트**(사업계획서/기능정의서/구현계획서/PRD/NDA 등)를 판단합니다.

### Step 2. N명 독립 리뷰어 실행 (서브에이전트 병렬 호출)

**N=3**을 기본값으로, 3명의 서브에이전트를 **병렬로** 호출합니다 (순차 실행 금지). 각 서브에이전트는:

- 동일한 리뷰 프롬프트 (`prompts/review.md`)를 받음
- 동일한 문서를 받음
- **서로의 결과를 모름** (독립성이 핵심)
- 자유 텍스트로 ISS-1, ISS-2, ... 형태의 결함 목록을 출력

#### 문서 전달 방식 — 파일 경로 우선 (기본값)

**기본 규칙: 항상 파일 경로만 전달하세요.** 서브에이전트가 `read_file` (또는 `Read`) 도구로 직접 파일을 읽습니다.

이유:
- 메인 에이전트가 3개 서브에이전트를 **병렬로 호출**할 때, tool call JSON 3개에 **문서 전문이 3번 중복 생성**됩니다. 큰 문서면 출력 토큰 한도를 넘겨 `InvalidJson`/truncation 에러가 납니다 (2026-04-28 실제 사고).
- 파일 경로는 수십 바이트, 문서 전문은 KB~MB. **경로 전달이 토큰을 1,000배 이상 절약**합니다.
- 서브에이전트가 `read_file`로 읽어도 각자의 컨텍스트 안에서만 쓰이고 부모 세션엔 누적되지 않아 메인 토큰도 절약됩니다.

**인라인 전달이 허용되는 예외**:
- 사용자가 **인라인 텍스트**(파일이 아닌 채팅 붙여넣기)로 문서를 제공했고,
- **AND** 문서가 500 단어 / 2KB **미만**일 때.

위 두 조건을 **모두** 만족하지 않으면 **반드시 파일로 먼저 저장한 뒤 경로를 전달**하세요. 파일명은 `/tmp/consensus-review-input-{YYYYMMDD-HHMMSS}.md` 같은 임시 경로를 사용해도 됩니다.

#### 프롬프트 변수 치환

서브에이전트에 전달할 내용:

- `{document_path}` ← **파일 시스템 경로** (기본). 예외 경우만 문서 전문 (500 단어/2KB 미만).
- `{language}` ← "ko" 또는 "en"
- `{doc_type_hint}` ← "사업계획서", "기능정의서", "구현계획서", "architecture", "general" 중 하나

**기타 주의**:
- 서브에이전트 3명을 **동시에** 실행 (병렬)
- 모델은 사용자의 현재 설정을 따릅니다 (스킬에서 지정 안 함)
- temperature는 서브에이전트 기본값 사용 (일반적으로 다양성 확보됨)

### Step 3. 집계 (본인 에이전트가 직접 수행)

3명의 raw 결과 + 원본 문서를 `prompts/aggregate.md` 프롬프트에 넣고 본인(메인 에이전트)이 집계합니다. 서브에이전트를 또 만들 필요 없습니다 — 집계는 1회 호출.

`prompts/aggregate.md`의 변수:

- `{n_agents}` ← 3
- `{document}` ← 원본 문서
- `{reviews}` ← 3명의 raw 출력 (각각 `=== Reviewer 1 ===\n{raw}\n\n=== Reviewer 2 ===\n{raw}\n\n=== Reviewer 3 ===\n{raw}` 형태로 이어붙임)

집계 결과는 🔴/🟡/⚪ Tier가 붙은 ISS-1, ISS-2 … 형태의 통합 목록입니다.

### Step 4. 사용자에게 보고 + 파일 저장

`examples/output_template.md`의 형식으로 최종 Markdown 리포트를 생성합니다. **파일 저장이 기본이고, 채팅 출력은 요약만 합니다.**

> 🚨 **Critical — 토큰 고갈 방지 규칙**
>
> 전체 리포트 본문은 **한 응답에 한 번만** 등장해야 합니다.
> `write.content` 안에만 전문을 넣고, 같은 턴의 `assistant_text`에는 **전문을 복제하지 마세요** (짧은 요약만).
> 전문을 두 번 출력하려 하면 `max_output_tokens`를 넘겨 write의 JSON이 잘리고 파일 저장이 실패합니다 (2026-04-28 실제 사고).
>
> **순서**:
> 1. 먼저 `write` 툴 호출 — `content` 인자에 전체 리포트 Markdown 전문
> 2. 같은 응답의 `assistant_text`에는 **짧은 요약만** (아래 4b 포맷)
> 3. 전문을 assistant_text에도 출력하는 행동은 **금지**

#### 4a. 파일로 전체 리포트 저장 (기본 동작)

리뷰 대상 문서와 **같은 디렉토리**에 다음 규칙으로 저장:

```
consensus-review-{원본파일명}-{YYYYMMDD-HHMMSS}.md
```

- `{원본파일명}`: 확장자 제외 (예: `requirements.md` → `requirements`)
- `{YYYYMMDD-HHMMSS}`: 실행 시점 기준 초 단위 타임스탬프 (같은 초에 겹칠 확률 ≈ 0)
- 예시: `consensus-review-requirements-20260426-152410.md`

**같은 문서를 여러 번 리뷰해도 타임스탬프가 달라 덮어쓰지 않습니다.** 리뷰 이력을 자동 보존합니다.

#### 4b. 채팅은 **요약만** 출력 (상세 덤프 금지)

파일 저장 후 채팅에는 **짧은 요약**만 출력하세요. 긴 Quote나 Reasoning 전체를 재출력하면 느리고 토큰을 낭비합니다.

**채팅에 출력할 것**:

```
✅ Consensus Review 완료

📄 저장 경로: ./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md

📊 요약
- 총 이슈: N개 (🔴 X, 🟡 Y, ⚪ Z)
- 리뷰어별 발견 수: R1 a건 / R2 b건 / R3 c건

🔴 High Confidence TOP 3 (제목만):
1. ISS-1: {title}
2. ISS-2: {title}
3. ISS-3: {title}

💡 전체 상세는 위 저장 경로의 파일을 열어보세요.
   "ISS-N 자세히" 라고 하면 특정 이슈만 펼쳐드릴 수 있습니다.
```

**금지**:
- 🔴/🟡/⚪ 전 이슈의 Quote와 Reasoning을 채팅에 다 출력하는 것 ❌
- 파일 내용을 그대로 복붙하는 것 ❌

**허용**:
- 사용자가 "ISS-5 자세히 보여줘"라고 **명시적으로 요청**했을 때 해당 이슈만 파일에서 읽어 보여주기.
- 사용자가 "전체 다 보여줘"라고 하면 그때 파일 내용 출력.

기본 동작은 **"파일 저장 + 요약 + 경로 안내"**입니다. 사용자가 터미널에서 파일을 열어 읽는 것이 훨씬 빠릅니다.

## 핵심 원칙 (위반 금지)

1. **독립성 유지** — Step 2의 서브에이전트는 서로의 결과를 절대 보면 안 됩니다. 프롬프트에 "다른 리뷰어의 결과" 같은 내용을 포함하지 마세요.
2. **원본 문서를 자르지 마세요** — 문서가 60K 토큰을 넘으면 요약이 아니라 청크 분할(나중에 확장 예정)을 쓰세요. 임의 잘림 금지.
3. **다수결 금지** — "3명 중 1명만 지적한 결함"이라도 ⚪ Tier로 살려두세요. 버리면 안 됩니다.
4. **집계 단계에서 "자기 finding 추가 금지"** — 집계 프롬프트에 이미 규칙이 명시되어 있습니다. 별도로 지시하지 말고 `prompts/aggregate.md`를 그대로 쓰세요.
5. **Tier 판정은 집계 LLM이** — 본인이 나중에 수동으로 🔴/🟡/⚪ 부여하지 마세요. `prompts/aggregate.md`가 이미 Tier 기준을 담고 있습니다.

## 설정 변경 (사용자가 요청하면)

- **N 변경**: "리뷰어 5명으로 해줘" → N=5. "2명만" → N=2 (비권장). 기본은 3.
- **언어 강제**: "한국어로 출력해줘" → 최종 리포트를 한국어로. 서브에이전트 프롬프트는 원본 유지.
- **특정 측면만**: "보안 이슈만 봐줘" → 리뷰 프롬프트에 지시 추가.

## 참고

- 설계 근거와 벤치마크 수치: `references/benchmark_summary.md`
- 리뷰 프롬프트: `prompts/review.md`
- 집계 프롬프트: `prompts/aggregate.md`
- 출력 템플릿: `examples/output_template.md`
- 샘플 실행 결과: `examples/sample_run.md` (테스트 후 업데이트)
