---
name: consensus-review
description: Review Korean/English documents for defects by running N=3 independent LLM reviewers as parallel subagents, then aggregate findings with consensus-tier labels (🔴 high / 🟡 medium / ⚪ low). Use when the user asks to 리뷰해줘, 문서 검토, 놓친 게 있는지 확인, 기능정의서 리뷰, 요구사항 검토, 사업계획서 검토, 구현계획서 검토, review a document, find defects, audit a spec, check for issues in a PRD / RFC / NDA / requirements / architecture / implementation plan. Trigger for filenames like requirements.md, spec.md, prd.md, design.md, architecture.md. NOT for: simple typo/grammar fixes, translation proofreading, documents under 500 words, 가벼운 편집 요청. The skill increases recall (catches more defects than a single reviewer) and provides tier-based priority for human review.
---

# Consensus Review

N명의 독립 서브에이전트 리뷰어를 병렬 실행해 문서 결함을 탐지하고 🔴/🟡/⚪ Tier로 분류합니다.

## 실행 단계

### Step 1. 문서 준비

1. 파일 경로가 주어지면 그대로 사용. 인라인 텍스트면 500 단어/2KB 미만일 때만 그대로, 초과 시 `/tmp/consensus-review-input-{YYYYMMDD-HHMMSS}.md`로 저장 후 경로 사용.
2. `{language}`: 문서 전반이 한글 문자 ≥50%면 "ko", 아니면 "en".
3. `{doc_type_hint}`: 파일명에 `prd|spec|requirements|nda|rfc|architecture` 매치 또는 본문 첫 200자에서 직접 매치되는 키워드 → 해당 유형. 매치 없으면 "general".

**예시**: `requirements.md` → "requirements"; 본문에 "1조(목적)", "비밀유지" → "NDA".

### Step 2. N명 독립 리뷰어 실행 (서브에이전트 병렬 호출)

> 🚨 **=== CRITICAL: INDEPENDENCE MODE — REVIEWERS MUST NOT SEE EACH OTHER ===**
>
> You are STRICTLY PROHIBITED from:
> - 한 리뷰어의 출력을 다른 리뷰어에게 전달
> - 리뷰 프롬프트에 다른 리뷰어를 언급
> - 리뷰어 호출을 직렬화하여 뒤 리뷰어가 앞 리뷰어 상태를 관찰
>
> 독립성이 이 스킬의 존재 이유입니다. 위반 시 이 스킬의 핵심 효과(확률적 커버리지로 +6.5~30.6%p 재현율 향상)가 사라집니다.

**N=3**을 기본값으로, 3명의 서브에이전트를 **병렬로** 호출합니다 (순차 실행 금지). 각 서브에이전트는:

- 동일한 리뷰 프롬프트 (`prompts/review.md`)를 받음
- 동일한 문서를 받음
- **서로의 결과를 모름** (독립성이 핵심)
- 자유 텍스트로 `ISS-N` 형태의 결함 목록을 출력. 각 항목은 최소 `Title / Type / Severity / Confidence / Evidence Strength / Quote(원문 그대로) / Reasoning` 필드를 포함. 상세 포맷과 앵커는 `prompts/review.md` 참조. **JSON을 강제하지 마세요** — JSON 파싱 에러가 나면 리뷰 결과가 통째로 날아갑니다 (2026-04-28 사고).

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
- 모델은 사용자의 현재 설정을 따릅니다 (스킬에서 지정 안 함). 최소 요구치: context window ≥ 200K. 그 미만 모델이 감지되면 사용자에게 알리고 중단.
- temperature는 서브에이전트 기본값 사용. 벤치마크(6 LLM × 3 benchmark) 기준 temperature=0.7~1.0 구간에서 재현율이 단조 증가. **0.3 미만으로 낮추지 마세요** — 독립성이 깨집니다.

#### 차단 상황 분기 (Blocked Approach)

실패/한계 상황에서 임의로 돌파하지 말고 아래 경로를 따르세요:

- **문서 > 60K 토큰**: 청크 분할은 아직 미구현. 사용자에게 알리고 중단: *"문서가 60K 토큰을 초과합니다(X 토큰). 현재 버전은 청크 분할을 지원하지 않습니다. 섹션을 나눠서 주시겠습니까?"* **임의로 자르거나 요약하지 마세요**.
- **`/tmp` 저장 실패** (디스크 풀/권한): 사용자에게 원인과 함께 보고하고 중단. **인라인 전달로 우회하지 마세요** — 토큰 한도를 넘겨 InvalidJson이 납니다.
- **서브에이전트 3명 중 1명이 실패/타임아웃**: 남은 2명으로 진행하되 리포트 상단에 `"Reviewer X 실패 — N=2로 집계됨"`을 명시. 실패한 리뷰어를 같은 프롬프트로 재시도하지 마세요 (차단 원인이 동일할 가능성이 높음).

### Step 3. 집계 (본인 에이전트가 직접 수행)

> 🚨 **REMEMBER**: By Step 3 the reviewers have finished independently. 메인 에이전트(당신)만이 세 개의 출력을 모두 볼 수 있습니다. 통합한 출력을 어떤 리뷰어에게도 "refinement"용으로 되돌려 보내지 마세요.

3명의 raw 결과 + 원본 문서를 `prompts/aggregate.md` 프롬프트에 넣고 본인(메인 에이전트)이 집계합니다. 서브에이전트를 또 만들 필요 없습니다 — 집계는 1회 호출.

`prompts/aggregate.md`의 변수:

- `{n_agents}` ← 3
- `{document}` ← 원본 문서
- `{reviews}` ← 3명의 raw 출력 (각각 `=== Reviewer 1 ===\n{raw}\n\n=== Reviewer 2 ===\n{raw}\n\n=== Reviewer 3 ===\n{raw}` 형태로 이어붙임)

집계 결과는 🔴/🟡/⚪ Tier가 붙은 ISS-1, ISS-2 … 형태의 통합 목록입니다.

> 🚨 **중요 — 집계 결과를 `assistant_text`로 출력하지 마세요.**
>
> 집계 결과(전체 이슈 목록)는 **내부 산출물**입니다. 채팅 응답 텍스트로 찍지 말고, **곧바로 Step 4a의 `write` 툴 호출의 `content` 인자로 전달**하세요.
>
> 이를 어기면 같은 리포트가 응답 텍스트와 `write.content`에 이중 출력되어 `max_output_tokens`를 넘기고 `InvalidJson`으로 잘립니다 (2026-04-28 실제 사고).

### Step 4. 사용자에게 보고 + 파일 저장

`examples/output_template.md`의 형식으로 최종 Markdown 리포트를 생성합니다. **파일 저장이 기본이고, 채팅 출력은 요약만 합니다.**

> 🔒 **불변식 (Invariant) — 반드시 지켜야 하는 규칙**
>
> **전체 리포트 본문은 한 응답에 오직 한 곳에만 등장합니다** — `write` 툴의 `content` 인자입니다.
>
> - `write.content` ← 전체 리포트 Markdown 전문 (✅ 여기 한 번)
> - `assistant_text` (채팅) ← **Step 4b의 짧은 요약 템플릿만** (✅ 요약만)
>
> 이 불변식을 지키면 출력 토큰이 절반으로 줄어 대형 리포트에서도 JSON이 잘리지 않습니다.
>
> **자주 하는 실수 — 이렇게 하지 마세요**:
>
> - ❌ assistant_text에 "집계 결과를 먼저 보여주고" 이어서 write 호출 — 본문이 두 번 등장
> - ❌ Step 4b의 요약에 이슈 전체 상세(Quote, Reasoning)를 포함 — TOP 3 **제목만**
> - ❌ "파일에 저장 예정입니다" 같은 선언 후 본문을 채팅에 먼저 찍기 — write가 먼저 호출돼야 함
>
> **순서**:
> 1. 먼저 `write` 툴 호출 — `content`에 전체 리포트 Markdown
> 2. 같은 응답의 `assistant_text`에는 Step 4b의 **짧은 요약 템플릿만**
> 3. Step 3 집계 결과를 중간에 채팅에 덤프하는 행동 **금지**

#### 4a. 파일로 전체 리포트 저장 (기본 동작)

이 단계에서 `write` 툴을 호출합니다. (불변식 재확인: `write.content`에만 전체 본문, `assistant_text`에는 Step 4b 요약만.)

리뷰 대상 문서와 **같은 디렉토리**에 다음 규칙으로 저장:

```
consensus-review-{원본파일명}-{YYYYMMDD-HHMMSS}.md
```

- `{원본파일명}`: 확장자 제외 (예: `requirements.md` → `requirements`)
- `{YYYYMMDD-HHMMSS}`: 실행 시점 기준 초 단위 타임스탬프 (같은 초에 겹칠 확률 ≈ 0)
- 예시: `consensus-review-requirements-20260426-152410.md`

**같은 문서를 여러 번 리뷰해도 타임스탬프가 달라 덮어쓰지 않습니다.** 리뷰 이력을 자동 보존합니다.

**저장 후 검증**: `write` 툴 반환값을 확인한 뒤 Step 4b 요약을 출력하세요. 실패하면 차단 상황 분기의 `/tmp 저장 실패` 항목을 따르세요 — **본문을 채팅에 대신 덤프하지 마세요**.

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

## 설정 변경 (사용자가 요청하면)

- **N 변경**: "리뷰어 5명으로 해줘" → N=5. "2명만" → N=2 (비권장). 기본은 3.
- **언어 강제**: "한국어로 출력해줘" → 최종 리포트를 한국어로. 서브에이전트 프롬프트는 원본 유지.
- **특정 측면만**: "보안 이슈만 봐줘" → 리뷰 프롬프트에 지시 추가.
