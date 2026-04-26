# 출력 템플릿 (메인 에이전트가 사용자에게 최종 보고 시 사용)

> 메인 에이전트가 집계 결과(Step 3)를 받은 뒤, 이 템플릿에 맞춰 사용자에게 최종 Markdown 리포트를 출력합니다.

---

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

### ISS-1: {제목}

- **Type**: {Inconsistency / Omission / Ambiguity / Terminology / Structural}
- **Severity**: {CRITICAL / MAJOR / MINOR}
- **Confidence**: {1-10}/10
- **Evidence Strength**: {1-5}/5
- **Quotes**:
  - 📍 "{exact quote}" — Reviewer 1
  - 📍 "{exact quote}" — Reviewer 2
  - 📍 "{exact quote}" — Reviewer 3
- **Reasoning**: {1-2 문장}

### ISS-2: ...

---

## 🟡 Needs Review — 다수 합의 ({Y}개)

> 3명 중 2명이 지적했습니다. 검토를 권장합니다.

### ISS-N: {제목}

- **Type**: {...}
- **Agents**: 2/3
- **Confidence**: {1-10}/10
- **Quotes**:
  - 📍 "{quote}" — Reviewer {id}
  - 📍 "{quote}" — Reviewer {id}
- **Reasoning**: {1-2 문장}

---

## ⚪ Low — 소수 지적 ({Z}개)

> 3명 중 1명이 지적했습니다. 시간 여유가 있을 때 확인하세요.

- **ISS-N**: {제목} — {한 줄 요약}
- **ISS-N+1**: {제목} — {한 줄 요약}

(상세 내용은 요청 시 펼쳐 드립니다.)

---

## 💡 권장 조치

- **🔴부터 우선 검토**하세요. 3명 전원 합의라 진짜 결함일 확률이 높습니다.
- **시간 여유가 있으면 🟡까지** 보세요. 다수 합의라 실제 정밀도도 중간 이상입니다.
- **⚪는 리소스 여유 시**에만. 1명만 본 것이라 환각(hallucination) 가능성이 상대적으로 큽니다.

---

📄 **Saved to**: `./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md`

---

## ℹ️ 이 스킬에 대해

이 리뷰는 **같은 모델로 N명의 독립 리뷰어를 병렬 실행 → 결과 합의 Tier로 분류** 하는 `consensus-review` 스킬에 의해 생성되었습니다.

단일 리뷰가 확률적으로 놓치는 결함들을 독립 N명이 교차 탐지해 **재현율을 6.5~30.6%p 끌어올리는** 것을 목표로 설계되었습니다 (6개 LLM × 3개 벤치마크 실험 근거).

- 🔴 정밀도 (실험치, FAVA 기준): 약 80~90%
- 🟡 정밀도: 약 30~40%
- ⚪ 정밀도: 약 15~20%

상세 근거: `references/benchmark_summary.md`
