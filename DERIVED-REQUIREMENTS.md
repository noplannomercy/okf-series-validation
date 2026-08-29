# 파생 요구사항 — Producer 행동 계약 (초안)

## 이 문서의 지위

| 항목 | 값 |
|---|---|
| 성격 | `OKF-RE-VALIDATION.md` 의 **확정된 실측 결과에서 파생된 요구사항 초안** |
| **아님** | 검증 리포트의 일부가 아니다. architecture 설계가 아니다. 채택된 요구사항이 아니다 |
| 상태 | **확인 후 확정 대상** — `REQUIRED` 로 표기하지 않는다 |
| 분리 사유 | 검증 리포트는 *직접 확인한 것*만 담는다. 아래 항목의 **내용**은 실측으로 확정되었으나, 이를 "우리 producer 가 반드시 만족해야 한다"고 **규정하는 행위** 자체는 검증이 아니라 결정이다. 그래서 리포트 밖으로 뺀다 |
| 작성일 | 2026-08-29 |
| 근거 문서 | `OKF-RE-VALIDATION.md` (동일 디렉토리) |

## 전제 (미확정)

- **`okf-oracle` integration = HOLD-ENV.** 실물이 이 PC에 없다. 회사 PC read-only 확인 전까지 그 의존 구조를 단정하지 않는다. 사용자 진술(`go.mod` 없음)은 **미검증 진술**로만 존재한다.
- 따라서 아래 항목이 **어느 구현체에 적용되는지는 아직 정해지지 않았다.**
- **"언어 무관"** 은 이번 검증이 담보하지 않는다. 실측은 전부 Go 구현(`okf-go` fork) 위에서 수행되었다. 다른 언어에서 동일 계약이 성립하는지는 검증하지 않았다.

## 왜 이 문서가 필요한가

`OKF-RE-VALIDATION.md` 부록 D·E 는 `okf-go` fork 가 경계 요구를 만족함을 실측했다. 그러나 최종 producer 가 그 Go 라이브러리를 물지 않을 가능성이 열려 있다(HOLD-ENV). 그 경우 fork 에서 넘어가는 것은 **라이브러리가 아니라 그것이 만족한 행동**이다.

이 문서는 그 행동을 한곳에 모아 둔다. 확정 여부는 별도 결정이다.

---

## 1. 경계 회귀 단언 14개

**출처**: `okf-skills-fork/okf-go/boundary_test.go` — 실측 확정
**상태**: fork 에서 **14/14 PASS**. 동일 스위트를 패치 없는 upstream 에 적용하면 **7개 함수 FAIL** (반증력 확인 완료)

producer 가 Go 가 아닌 경우, 이 단언들은 해당 구현 언어의 회귀 테스트로 옮겨야 의미가 유지된다. 단언 목록은 파일 참조(B1, B2, B3, B4, B5, B6, B6b, B7, B8, B9, B9b, B10 계열, B11 계열, B12).

## 2. 동작 규칙 P1~P5

**출처**: `okf-skills-fork/FORK.md` — 각 규칙에 SPEC 조항 근거 명시, 실측 확정

| # | 규칙 | 근거 |
|---|---|---|
| P1 | `stale_after` 는 정본 v0.2 **절대 시각**을 수용하고, 파싱 실패는 **fail-closed(stale)** 로 처리한다 | SPEC §5.5 |
| P2 | 형태가 어긋난 `verified` 는 무성 소멸시키지 않고 **오류로 노출**한다 | §5.2 의 검증 기록 보존 취지 |
| P3 | 모델링되지 않은 frontmatter 키를 **왕복 보존**한다 | SPEC §4.1 *"consumers SHOULD preserve unknown keys when round-tripping"* |
| P4 | 구조 변경 재생성 시 `verified` / `sources` / `usage_window` / `status` / `stale_after` 를 **이월**한다 | SPEC §5.2 *"content can change without re-confirmation"* |
| P4b | 확장 메타데이터는 **키 단위로 병합**한다 (existing + fresh, 충돌 시 fresh 우선). 컬렉션 전체 치환에 의존하지 않는다 | 부록 H / C5 실측 |
| P4c | 검증 이력은 **병합**한다 — existing + fresh, 동일 `(by, at)` 중복 제거, 안정 연결 순서 | SPEC §5.2 *"Multiple entries capture independent checks"* / 부록 I |
| P5 | 구조 해시에 **신원**(`type` / `title` / `resource`)을 포함해, body 가 동일한 채 자산 신원만 바뀌어도 변경으로 판정한다 | 무성 신원 훼손 차단 (부록 A.1) |

**적용 순서 주의**: P4 를 P5 보다 먼저 적용해야 한다. P5 단독 적용 시 해시 무효화로 발생하는 1회 전면 재작성이 신뢰·provenance 를 파괴한다. 실측 근거는 `FORK.md` 및 부록 D.5.

## 3. §4c 신뢰 기록 정책

**출처**: `okf-skills-fork/skills/okf-enrich/SKILL.md` — 실측 확정 (부록 F)

- 보강 수행 주체는 `generated.by` 에 기록한다 (SPEC §5.2: `generated` = *how the content was produced*, `generated.by` REQUIRED)
- **보강 주체를 `verified` 에 기록하지 않는다.** `verified` 는 *소스에 대해 확인한* 주체의 기록이며, SPEC §5.2 는 두 역할을 분리한다
- 독립 확인자에 의한 `verified` 기록과 기존 기록 보존은 유지한다

**결정적 검증 결과**: 자기서명 상태 → `machine-confirmed` (baseline) / 패치된 경로 → `unverified` / 독립 확인 경로 3종 전부 기존 동작 유지.

## 4. self-sign 기계적 탐지 게이트

**상태**: **구현됨 (부록 G)** — `okf-lint -policy-no-self-sign`, opt-in, SPEC conformance 와 분리된 policy 층

§4c 는 instruction 이므로 지시 위반 자체를 막지는 못하지만, 그 결과로 생긴 `generated.by == verified.by` 상태는 이제 결정적으로 탐지된다. 게이트는 opt-in 이며 검사 시점에만 작동한다 — 작성 시점 차단이 아니다. producer 가 Go 가 아닌 경우 동일 정책을 해당 검증 계층에 옮겨야 한다.


---

## 5. Uncertainty / evidence-status policy (F1 + F2 + F5)

**상태**: 요구사항 정의 단계. 코드 변경 없음 — `okf-viz`·`okf-enrich` 무수정.
**대상**: F1(근거 부족 시 보류 경로 부재) + F2(충돌 evidence 처리 부재) + F5(관측/추론 표현 경로 부재) 를 하나의 정책으로 묶는다.

> **PROVISIONAL 표기.** 아래 fixture 의 키 이름 `x_gcore_evidence_status` 와 값 `inferred` 는 **측정용 임시값**이다. production key 이름·schema·state vocabulary 는 이번 단계에서 확정하지 않는다.

### 5.1 Observed behavior (실측)

fixture C1~C5, `okf-go` fork @ `b79bee7` 기준.

| ID | 상태 | 결과 |
|---|---|---|
| **C1** | 정상 description, 마커 없음 | enriched 로 계수 |
| **C2** | 정상 description + **uncertainty 마커** | **enriched 로 계수** — 마커는 무시된다 |
| **C3** | placeholder description, 마커 없음 (대조군) | placeholder 로 계수, `enrich-first` 에 등장 |
| 집계 | — | `concepts=3, enriched=2, placeholders=1, enriched_pct=66.7%`, **`enrich-first` 에 C2 없음** |
| conformance | C2 의 마커가 concept 문서에 존재 | **위반 0건** — 확장 키는 SPEC 위반이 아니다 (SPEC §11: *unknown additional frontmatter keys* 로 거부 금지) |
| **C4** | 마커 보유 concept 를 **구조 변경 재생성** 경로에 통과 (connector fresh 에 확장 키 없음) | **보존됨** (`changed=true`, 마커 유지) |
| **C5** | 동일하되 **fresh 가 확장 키를 하나라도 보유** | **소실** — 기존 `Extra` 전체가 fresh 의 `Extra` 로 대체 |

**해석**: `IsPlaceholderDescription` 은 description 문자열만 정규식으로 판정하며 frontmatter 확장 메타데이터를 보지 않는다(부록 B 에서 이미 확인된 구조). 따라서 uncertainty 상태의 concept 가 **완료된 보강과 구분되지 않고**, 재작업 대상 목록(`enrich-first`)에도 오르지 않는다.

**이것을 upstream 결함으로 규정하지 않는다.** 현재 지표의 **정의 한계**다. `okf-viz coverage` 는 *"how much is enriched"* 를 세겠다고 명시했고 그대로 동작한다.

### 5.2 Policy requirement (F1 + F5)

1. 근거가 부족하거나 추론에 의존한 concept 를 **"정상 enrichment 완료"로 계수하지 않을 수 있는 명시적 HOLD/UNCERTAIN 상태**가 필요하다.
2. 이 상태는 **OKF core SPEC 필드를 임의로 변경하지 않고** 확장 메타데이터로 표현하는 방안을 우선 후보로 둔다.
3. **불린으로 축약하지 않는다.** F1 은 *쓸 것인가*(보류), F5 는 *쓴 것을 어떻게 표시할 것인가*(관측 vs 추론)로 서로 다른 질문이다. 두 요구를 표현하려면 **최소한 다중 상태를 표현할 수 있어야 한다**. 구체적 state vocabulary 는 확정하지 않는다.
4. 마커는 **재생성·구조 변경 후에도 보존되어야 한다.**
5. **배치 제약 — 마커는 concept 문서에만 둔다.**
   - 루트 `index.md`: `okf_version` 외 키는 conformance 규칙 **`root-index-extra-keys`** 위반
   - 하위 디렉토리 `index.md`: 규칙 **`subdir-index-frontmatter`** 상 frontmatter 자체를 가질 수 없음
   - `log.md`: 예약 파일
   ⇒ 번들 전역 상태로 둘 수 있는 자리는 없다.

### 5.3 Invariant (계산식 대신 고정하는 것)

> **Uncertainty / HOLD 상태가 aggregate metric 에서 보이지 않게 사라져서는 안 된다.**

- numerator / denominator treatment 는 **PROVISIONAL / UNDECIDED**.
- *분자만 제외* 와 *분자·분모 모두 제외* 중 **어느 것도 이번 단계에서 채택하지 않는다.**
- 사유: 이 선택은 현재 evidence 에서 도출되는 사항이 아니라, HOLD 를 업무적으로 "미완료"로 볼지 "집계 대상 제외"로 볼지 결정하는 **policy semantics** 다.
- 다만 **어떤 계산식을 택하더라도 HOLD/UNCERTAIN 의 count 또는 rate 가 별도로 관측 가능해야 한다.**

### 5.4 확장 메타데이터 사용 가능성 — 근거와 제약

**가능성 근거 (P3)**: 부록 D 에서 `Frontmatter.Extra` (`yaml:",inline"`) 로 미모델링 키의 왕복 보존이 실측 확정되었다. 부록 D.4 end-to-end 에서 실제 `okf-fs` 바이너리로 `gcore_evidence_id` 가 구조 변경 재생성을 통과한 **선행 evidence** 가 있으며, 본 단계의 C4 는 uncertainty 의미의 임시 마커로 **동일 메커니즘을 재확인**한 것이다(신규 발견이 아니다).

**제약 (C5 실측)**: P4 의 이월 로직은 **all-or-nothing** 이다 — `merged.Extra` 가 비어 있을 때만 `existing.Extra` 를 이월한다. 따라서 connector 가 **확장 키를 하나라도 방출하는 순간 기존 `Extra` 전체가 대체되고 uncertainty 마커가 소실된다.**

⇒ 당시 "마커가 재생성을 견딘다"는 보장은 **어떤 connector 도 확장 키를 쓰지 않는다는 우연**에 의존했다.

⇒ **해소됨 (부록 H / P4b).** `okf-go` fork 의 `MergeConcept` 이 확장 키를 키 단위로 병합하도록 정정되었고, C5-1/2/3 이 boundary suite 의 정식 회귀 케이스로 편입되었다 (Producer boundary gate 14 → 15).

⇒ **요구사항 (producer 무관)**: 확장 메타데이터의 병합은 **키 단위 보존**이어야 한다. 컬렉션 전체 치환에 의존하면 안 된다.

⇒ **네임스페이스 제약**: **Producer-owned extension keys and agent/policy-owned extension keys must not share the same key namespace unless explicit ownership/precedence is defined.** 현재의 `fresh wins` 는 **충돌 fallback 규칙이지 ownership model 이 아니다.** production namespace/schema 자체는 아직 확정하지 않는다.

### 5.5 기존 coverage 와 enterprise accepted/trusted coverage 의 분리

self-sign 에서 적용한 것과 동일한 원칙을 유지한다 — **기본 semantics ≠ enterprise policy**.

| 층 | 내용 |
|---|---|
| **SPEC / 기본 coverage** | 기존 의미 그대로 유지. `okf-viz coverage` 의 정의를 바꾸지 않으며, 마커 무시를 결함으로 주장하지 않는다 |
| **Enterprise acceptance policy** | uncertainty/evidence-status 마커가 활성인 concept 를 **accepted/trusted enrichment 로 간주하지 않는다.** 별도 metric/gate 에서 처리하며, 이를 SPEC conformance 나 upstream coverage 정의로 주장하지 않는다 |

### 5.6 미확정 항목 (이번 단계에서 정하지 않음)

- production key 이름
- schema / state vocabulary (다중 상태 필요라는 제약만 확정)
- producer 구현 위치
- policy metric / gate 의 구현 위치
- numerator / denominator treatment (§5.3 invariant 만 고정)

### 5.7 F2 — conflicting evidence (F1/F5 정책의 확장)

**Observation 은 신규가 아니다.** 부록 C.0 에서 이미 확정되었다 — `okf-enrich` 에 ①충돌 evidence 처리 절차가 없고 ②unresolved/conflict 상태를 표현할 필드·절차가 없다. 본 절은 그 관측을 재도출하지 않고 **정책 요구사항으로 통합**한다.

**Policy invariant (F2)**

> 서로 모순되는 evidence 가 존재하고 현재 evidence 만으로 어느 쪽이 우세한지 결정할 수 없다면, **한쪽으로 조용히 수렴한 서술을 생성해서는 안 된다.**

그 경우:
- unresolved/conflict 상태를 **명시적으로 유지**할 수 있어야 한다
- concept 를 **HOLD/UNCERTAIN 계열 상태로 취급**할 수 있어야 한다
- **충돌하는 양쪽 evidence/source 를 모두 보존**해야 한다
- conflict 가 해소되기 전까지 **enterprise acceptance/trusted enrichment 에서 정상 확정 상태처럼 취급되어서는 안 된다**

**vocabulary 요구**: §5.2-3 의 multi-state 제약에 **conflict 상태를 표현할 수 있어야 한다**는 요구를 추가한다. fixture 의 `conflicting` 은 **PROVISIONAL 측정값**이며 최종 vocabulary 가 아니다.

**§5.3 invariant 승계 (변경 없음)**: **HOLD / UNCERTAIN / CONFLICT 상태가 aggregate metric 에서 보이지 않게 사라져서는 안 된다.** numerator/denominator treatment 는 계속 **PROVISIONAL / UNDECIDED**. 기본 `okf-viz coverage` semantics 도 변경하지 않는다.

**제약 — `sources` 보존은 conflict 기록이 아니다**
OKF `sources` 리스트는 *어느 주장을 어느 source 가 뒷받침하는지*(§5.1 의 `id` 는 각주 귀속용)도, *둘이 서로 모순이라는 사실*도 표현하지 않는다. 따라서 **충돌 관계는 evidence-status 마커가 지고 `sources` 는 지지 못한다.** 양쪽 source 를 보존하는 것은 필요조건이지 충분조건이 아니다.

**F2 최소 확인 (실측)**

| Fixture | 결과 |
|---|---|
| **F2-C1** 모순되는 source 2개 + 정상 description + concept-level 마커(`conflicting`) | SPEC conformance 위반 **0건** / 기존 coverage 에서 **enriched 로 계수**(`enriched_pct=100.0%`, `enrich-first` 비어 있음) |

⇒ **coverage 결함으로 기록하지 않는다.** 기본 coverage 는 description/placeholder **진행률 지표**이고, conflict-aware acceptance 는 **enterprise policy layer 의 책임**이다.

| Fixture | 결과 |
|---|---|
| **F2-C2 (a)** fresh 에 `sources` 없음 | 마커 생존, **양쪽 source 생존** |
| **F2-C2 (b)** fresh 가 자체 `sources` 방출 | 마커 생존, **양쪽 source 소실** — connector 의 source 1개로 대체 |
| **F2-C2 (c)** fresh 가 자체 확장키 방출 | 마커 생존, 양쪽 source 생존 |

마커 생존 자체는 **C5/P4b 와 동일 메커니즘의 재확인**이며 P4b 의 신규 기능이 아니다.

### 5.8 Boundary candidate (미해결, 이번 단계에서 고치지 않음)

**대상 = `Sources` 하나.** 범위 측정 완료(부록 H 추기 2) 후 `Verified` 는 P4c(부록 I)로 해소되었다.

`MergeConcept` 은 해당 필드가 fresh 에서 비어 있을 때만 existing 을 이월한다. connector 가 값을 하나라도 방출하면 **기존 컬렉션 전체가 대체**된다.

| 필드 | fresh 가 값을 들고 올 때 | 분류 |
|---|---|---|
| ~~`Verified`~~ | 이력 병합 + `(by, at)` 중복 제거 | **해소됨 (P4c, 부록 I)** — 범위 밖 |
| `Sources` | 기존 provenance 전체 대체 | ALL-OR-NOTHING silent-loss |
| `Tags` | 합집합 | safe merge — **범위 밖** |
| `Extra` | 키 단위 병합 | safe merge (P4b) — **범위 밖** |
| `UsageWindow`/`Status`/`StaleAfter` | fresh 값 채택 | intentional fresh overwrite — 별도 정책 판단 |

F2 의 *"양쪽 evidence 보존"* 요구는 `Sources` 경로가 열려 있는 한 성립하지 않는다. (§3 의 §4c 신뢰 정책은 P4c 로 재생성 후에도 보장된다.)

**선행 조건 — 동일성 규칙이 먼저다.** 맵(`Extra`)과 달리 리스트는 병합 전에 "같은 항목"의 정의가 필요하다.
- `SourceEntry`: `Resource` 는 REQUIRED 이나 같은 리소스를 다른 관점으로 두 번 인용할 수 있고 `ID` 는 optional

동일성 규칙을 잘못 고르면 provenance 가 조용히 합쳐지거나 중복된다. **본 단계에서는 범위 측정까지만 하고 병합 규칙을 만들지 않았다.**

**게이트 사각지대**: `boundary_test.go` 의 B5·B6 는 fresh 가 비어 있는 방향만 검사한다. 위 실패는 **15개 게이트를 통과한 채로 발생한다.** 게이트 확장은 동일성 규칙 확정 이후에 한다.

### 5.9 이번 단계에서 하지 않은 것

`okf-go` 수정 없음 · `okf-viz` 수정 없음 · `okf-enrich` 수정 없음 · `okf-lint` 수정 없음 · policy metric 구현 없음 · 새 frontmatter schema/key/vocabulary 확정 없음 · numerator/denominator 결정 없음 · F6/F7 로 확장 없음 · architecture 설계 없음 · connector 추가 검증 없음.

---

## 미해결 상태 승계

아래는 `OKF-RE-VALIDATION.md` 부록 C 에서 확정된 미해결 항목이며, 본 문서가 상태를 바꾸지 않는다.

- F1 근거 부족 시 보강 보류 경로 부재 → **policy requirement defined / implementation not started** (§5)
- F2 충돌 evidence 처리 절차 부재 → **policy requirement defined / implementation not started** (§5.7)
- F5 관측/추론 미분리 → **policy requirement defined / implementation not started** (§5)
- F6 `verified` 중복 제거 규칙 부재 → **REMEDIATED (merge boundary 한정)** — 병합 경계에서 자가 치유. 병합 사이 직접 기록된 중복은 다음 재생성까지 잔존 (부록 I.5)
- F7 이전 추론 철회·하향 규칙 부재

`okf-enrich` 자산 판정은 **EXTEND 유지**. F1/F2/F5 는 요구사항이 정의되었을 뿐 구현은 착수되지 않았으며, F6/F7 은 상태 변경 없음. `REMEDIATED`/`CLOSED`/`IMPLEMENTED` 로 표기하지 않는다.
