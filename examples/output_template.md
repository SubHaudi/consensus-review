# 출력 템플릿 — 파일 + 채팅 요약 규칙

> 메인 에이전트가 집계 결과(Step 3)를 받은 뒤 두 곳에 결과를 남깁니다:
>
> - **파일 (전체 상세 리포트)**: `consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md`
> - **채팅 (짧은 요약)**: 저장 경로 + 요약만 출력. 긴 Quote 덤프 금지.

---

## A. 파일에 저장할 전체 리포트 형식

파일 내용은 아래 구조를 따릅니다. 이것이 "상세 리포트"입니다.

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

### ISS-N: {제목}
... (🔴와 동일 포맷)

---

## ⚪ Low — 소수 지적 ({Z}개)

- **ISS-N**: {제목} — {한 줄 요약}
- **ISS-N+1**: {제목} — {한 줄 요약}

---

## 💡 권장 조치

- 🔴부터 우선 검토하세요. 3명 전원 합의라 진짜 결함일 확률이 높습니다.
- 시간 여유가 있으면 🟡까지. 다수 합의라 정밀도 중간 이상입니다.
- ⚪는 리소스 여유 시에만. 환각(hallucination) 가능성이 상대적으로 큽니다.
```

---

## B. 채팅에 출력할 **짧은 요약** 형식

파일 저장 후 채팅에는 아래만 출력하세요. 전체 상세는 **절대 채팅에 덤프하지 마세요**.

```
✅ Consensus Review 완료

📄 저장 경로: ./consensus-review-{원본}-{YYYYMMDD-HHMMSS}.md

📊 요약
- 총 이슈: N개 (🔴 X, 🟡 Y, ⚪ Z)
- 리뷰어별 발견 수: R1 a건 / R2 b건 / R3 c건

🔴 High Confidence TOP 3 (제목만):
1. ISS-1: {짧은 제목}
2. ISS-2: {짧은 제목}
3. ISS-3: {짧은 제목}

💡 전체 상세는 위 저장 경로의 파일을 열어보세요.
   "ISS-N 자세히"라고 하면 특정 이슈만 펼쳐드릴 수 있습니다.
```

### 왜 짧게 하는가

- 채팅에 전체 상세를 출력하면 토큰/시간이 2배 듭니다 (파일 + 채팅 양쪽 동시).
- 사용자가 상세를 읽는 가장 빠른 방법은 **에디터로 파일 열기**입니다.
- 필요한 이슈만 **on-demand**로 펼쳐 보여주는 게 UX 효율적입니다.

### 사용자가 "상세 보여줘"라고 명시적으로 요청한 경우

- "ISS-5 자세히" → 파일에서 해당 이슈만 읽어 채팅에 출력
- "전체 상세" → 파일 내용 전체 출력 (느림, 사용자가 명시적으로 원할 때만)
- 그 외엔 항상 요약 + 경로만.
