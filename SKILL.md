---
name: consensus-review
description: Review documents for defects by running N independent LLM reviewers as subagents, then aggregate findings with consensus-tier labels (🔴 high / 🟡 medium / ⚪ low). Use this skill when the user asks to 리뷰해줘, 문서 검토, 놓친 게 있는지 확인, review a document, find defects, audit a spec, check for issues in a business plan / feature spec / implementation plan / requirements document / technical design doc. Especially useful for 사업계획서, 기능정의서, 구현계획서, PRD, NDA, RFC, or any document where missing a defect has a real cost. The skill increases recall (catches more defects than a single reviewer) and provides tier-based priority for human review.
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

**N=3**을 기본값으로, 3명의 서브에이전트를 병렬로 호출합니다. 각 서브에이전트는:

- 동일한 리뷰 프롬프트 (`prompts/review.md`)를 받음
- 동일한 문서를 받음
- **서로의 결과를 모름** (독립성이 핵심)
- 자유 텍스트로 ISS-1, ISS-2, ... 형태의 결함 목록을 출력

서브에이전트에게 전달할 프롬프트는 `prompts/review.md`를 읽어 다음 변수를 치환하세요:

- `{document}` ← 사용자가 준 문서 전체 (자르지 말 것)
- `{language}` ← "ko" 또는 "en"
- `{doc_type_hint}` ← "사업계획서", "기능정의서", "구현계획서", "general" 중 하나

**중요**:
- 서브에이전트 3명을 **동시에** 실행하세요 (순차 실행 금지 — 독립성이 깨지지 않지만 시간 낭비)
- 모델은 사용자의 현재 설정을 따릅니다 (스킬에서 지정 안 함)
- temperature는 서브에이전트가 기본값을 쓰면 됩니다 (일반적으로 다양성이 확보됨)

### Step 3. 집계 (본인 에이전트가 직접 수행)

3명의 raw 결과 + 원본 문서를 `prompts/aggregate.md` 프롬프트에 넣고 본인(메인 에이전트)이 집계합니다. 서브에이전트를 또 만들 필요 없습니다 — 집계는 1회 호출.

`prompts/aggregate.md`의 변수:

- `{n_agents}` ← 3
- `{document}` ← 원본 문서
- `{reviews}` ← 3명의 raw 출력 (각각 `=== Reviewer 1 ===\n{raw}\n\n=== Reviewer 2 ===\n{raw}\n\n=== Reviewer 3 ===\n{raw}` 형태로 이어붙임)

집계 결과는 🔴/🟡/⚪ Tier가 붙은 ISS-1, ISS-2 … 형태의 통합 목록입니다.

### Step 4. 사용자에게 보고

`examples/output_template.md`의 형식으로 최종 Markdown 리포트를 사용자에게 출력하세요. 내용에는 다음이 반드시 포함되어야 합니다:

- **요약 라인**: "총 N개 고유 이슈 (🔴 X, 🟡 Y, ⚪ Z) / Reviewer1: a건, Reviewer2: b건, Reviewer3: c건"
- **🔴 High Confidence 섹션**: 3명 전원 합의. 상세.
- **🟡 Needs Review 섹션**: 다수 합의(2/3). 상세.
- **⚪ Low 섹션**: 소수 지적(1/3). 요약만.
- **권장 조치 문장**: "🔴부터 우선 검토하세요. 시간 여유가 있으면 🟡까지 보는 것을 권장합니다."

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
