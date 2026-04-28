# 출력 템플릿 — 파일 전체 리포트 + 채팅 요약 분리

> 메인 에이전트가 집계 결과(Step 3)를 받은 뒤 두 곳에 결과를 남깁니다:
>
> - **파일 (전체 상세 리포트)**: `consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md`
> - **채팅 (짧은 요약)**: 저장 경로 + 요약만 출력.

> 🔒 **불변식 (Invariant)**
>
> 전체 리포트 본문은 한 응답에 **오직 한 곳에만** 등장합니다 — `write.content`.
> `assistant_text`(= 채팅)에는 B 섹션의 **리터럴 템플릿**만 들어갑니다.
> 이 불변식 위반 시 `max_output_tokens`를 넘어 **InvalidJson 크래시가 발생하고 세션이 종료됩니다** (2026-04-28 실제 사고).

---

## === RECOGNIZE YOUR OWN RATIONALIZATIONS ===

당신은 아래 핑계로 B 템플릿을 벗어나려는 충동을 느낄 것입니다. **전부 무시하세요**:

- ❌ "사용자가 바로 볼 수 있게 🔴 이슈 본문도 채팅에 넣자" — 금지. 제목만.
- ❌ "Quote 1~2개 정도는 채팅에 넣어도 안전하지 않을까" — 금지. 제목만.
- ❌ "파일 쓰기가 실패할 수도 있으니 채팅에도 백업으로 넣자" — 금지. 실패하면 재시도하거나 실패를 보고.
- ❌ "사용자가 요약을 원하면 상세도 원할 것이다" — 금지. 명시적 "상세" 요청 전엔 경로만.

**B 템플릿 외의 모든 본문 복제는 REJECT 대상입니다.**

---

## A. 파일에 저장할 전체 리포트 형식

`write.content`에 담을 내용. 상세 리포트 전문은 여기에만.

### 치환 규칙

- `{중괄호}`는 실제 값으로 치환. 중괄호 자체는 출력 금지.
- 이모지/레이블(📋, 🔴, 🟡, ⚪, 💡)은 **리터럴**. 번역·축약·삭제 금지.
- 섹션 순서와 빈 줄 위치는 유지.

**Good**: `"📄 저장 경로: ./consensus-review-foo-20260428-010700.md"`
**Bad (플레이스홀더 잔존)**: `"📄 저장 경로: ./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md"`
**Bad (리터럴 변경)**: `"📄 Path: ./consensus-review-foo-20260428-010700.md"`

### 파일 템플릿

```markdown
# 📋 Consensus Review 결과

**문서**: {문서 이름 또는 간략 요약}
**리뷰어 수**: {N}명 독립 (Consensus Review 스킬)
**총 고유 이슈**: {총 개수}개 (🔴 {X}, 🟡 {Y}, ⚪ {Z})

| 리뷰어 | 발견한 결함 수 |
|---|---:|
| Reviewer 1 | {a}건 |
| Reviewer 2 | {b}건 |
| Reviewer 3 | {c}건 |

---

## 🔴 High Confidence — 전원 합의 ({X}개)

> 3명 전원이 지적한 이슈입니다. 우선 처리 대상입니다.

### ISS-1: {제목 ≤12 단어}

- **Verdict**: 🔴
- **Type**: {Inconsistency / Omission / Ambiguity / Terminology / Structural}
- **Severity**: {CRITICAL / MAJOR / MINOR}
- **Agents**: {count}/{N} flagged
- **Confidence**: {median}/10 (R1: {x}, R2: {y}, R3: {z})
- **Evidence Strength**: {median}/5
- **Quotes**:
  - 📍 "{exact quote}" — Reviewer 1
  - 📍 "{exact quote}" — Reviewer 2
  - 📍 "{exact quote}" — Reviewer 3
- **Reasoning**: {≤40 words}

### ISS-2: ...

---

## 🟡 Needs Review — 다수 합의 ({Y}개)

🔴와 동일 포맷. `Verdict`만 🟡로 변경.

---

## ⚪ Low — 소수 지적 ({Z}개)

- **ISS-N**: {제목} — {한 줄 요약}
- **ISS-N+1**: {제목} — {한 줄 요약}

---

## 💡 권장 조치

- 🔴부터 우선 검토하세요. 3명 전원 합의라 진짜 결함일 확률이 높습니다.
- 시간 여유가 있으면 🟡까지. 다수 합의라 정밀도 중간 이상입니다.
- ⚪는 리소스 여유 시에만. 환각 가능성이 상대적으로 큽니다.
```

---

## B. 채팅에 출력할 **짧은 요약** 형식

`assistant_text`에 담을 내용. **B 템플릿의 리터럴만** 출력. 전체 상세 복제 금지.

### 채팅 템플릿

```
✅ Consensus Review 완료

📄 저장 경로: ./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md

📊 요약
- 총 이슈: {N}개 (🔴 {X}, 🟡 {Y}, ⚪ {Z})
- 리뷰어별 발견 수: R1 {a}건 / R2 {b}건 / R3 {c}건

🔴 High Confidence TOP 3 (각 제목 ≤15 단어, Quote·Reasoning 금지):
1. ISS-1: {제목 ≤15 단어}
2. ISS-2: {제목 ≤15 단어}
3. ISS-3: {제목 ≤15 단어}

💡 전체 상세는 위 저장 경로의 파일을 열어보세요.
   "ISS-N 자세히"라고 하면 특정 이슈만 펼쳐드릴 수 있습니다.
```

### === 출력 예시 ===

**Good** (채팅 출력이 이래야 합니다):

```
✅ Consensus Review 완료

📄 저장 경로: ./consensus-review-plan_v2-20260428-010700.md

📊 요약
- 총 이슈: 14개 (🔴 3, 🟡 6, ⚪ 5)
- 리뷰어별 발견 수: R1 7건 / R2 9건 / R3 6건

🔴 High Confidence TOP 3 (각 제목 ≤15 단어, Quote·Reasoning 금지):
1. ISS-1: 인증 플로우와 스펙 §3.2 불일치
2. ISS-2: 에러 코드 409 정의 누락
3. ISS-3: 용어 "세션"이 두 의미로 혼용

💡 전체 상세는 위 저장 경로의 파일을 열어보세요.
   "ISS-N 자세히"라고 하면 특정 이슈만 펼쳐드릴 수 있습니다.
```

**Bad** (절대 금지 — InvalidJson 크래시 유발):

- 채팅에 🔴 이슈의 Quote·Reasoning 본문을 함께 출력 ❌
- 채팅에 `## 🔴 High Confidence — 전원 합의` 같은 **파일 템플릿 헤더** 출력 ❌
- `write.content`에 넣은 본문을 `assistant_text`에도 복제 ❌

**Bad (리터럴 훼손)**:

- `"Consensus Review 완료했습니다 ✅"` (순서·문구 변경)
- `"📄 파일: ..."` ("저장 경로" 리터럴 훼손)

---

## === 실패 및 분기 ===

- **파일 쓰기 실패** → 재시도 1회. 실패하면 채팅에 `"❌ 파일 저장 실패: {에러}. 저장 없이 진행할까요?"`만 출력. **본문을 채팅에 덤프 금지** — InvalidJson이 납니다.
- **🔴 0개** → `"🔴 High Confidence: 없음"` 한 줄. TOP 3 섹션 생략.
- **총 이슈 0개** → B 템플릿 대신 `"✅ Consensus Review 완료 — 지적된 이슈 없음. 파일: {경로}"`만.
- **사용자가 "상세 보기" 명시** (Good 표현: "ISS-5 자세히", "ISS-5 펼쳐", "전체 상세", "full report", "다 보여줘") → 해당 이슈만 채팅에 출력.
- **상세 요청 아님** (Bad 표현: "뭐가 중요해?", "요약 다시", "TOP 5로 늘려") → B 템플릿 범위 내 조정.
- **애매하면** → `"파일을 열어보시겠어요, 아니면 특정 ISS-N을 펼쳐드릴까요?"`로 되묻기.

---

## REMEMBER

파일(A)과 채팅(B)은 **두 개의 완전히 다른 출력**입니다. 같은 본문을 양쪽에 절대 복제하지 마세요. `write.content`에는 A 템플릿, `assistant_text`에는 B 템플릿 리터럴 — 이 매핑을 어기면 **InvalidJson 크래시로 세션이 실패**합니다.
