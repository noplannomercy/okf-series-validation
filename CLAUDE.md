# OKF 재검증 — 세션 컨텍스트

이 파일은 **새 세션이 처음부터 다시 파악하지 않도록** 하기 위한 것이다. 2026-08-29 하루치 작업의 상태와 규칙이 들어 있다.

## 지금 당장 할 일

**`okf-oracle` 에 `go.mod` 가 있는지 read-only 로 확인한다.** (30초)

- **있으면** → wiring 문제. `go.work` 에 fork 경로 추가로 끝난다
- **없으면** → 계약 이식 문제. `DERIVED-REQUIREMENTS.md` 의 항목을 해당 구현 언어로 옮겨야 한다

같이 볼 것: `go.work` / `replace` / import path / 빌드가 실제로 선택하는 `okf-go`.

**이 결과가 남은 작업 전체의 범위를 정한다.** 그전까지 okf-oracle 관련 상태는 `HOLD-ENV` 이며, **추측하지 않는다.**

## 저장소 구성

| | |
|---|---|
| 이 저장소 | 검증 리포트 · 파생 요구사항 (private, 내부 명칭 포함) |
| 코드 | `github.com/noplannomercy/okf-skills-fork` (private) |
| upstream | `github.com/xSAVIKx/okf-skills` @ `9740e89` |
| spec 정본 | `github.com/GoogleCloudPlatform/open-knowledge-format` @ `ad30107` (SPEC.md v0.2) |

저장소를 나눈 이유: upstream 에 PR 을 열 가능성이 있고, PR 은 브랜치 단위라 내부 문서가 같은 저장소에 있으면 diff 에 딸려 나간다.

## 무엇이 끝났나

`okf-go` fork **CLOSED**. 패치 P1~P5 + P4b + P4c, 2파일 규모. 회귀 게이트:

- **Producer boundary gate 16/16** (`okf-go/boundary_test.go`)
- **Trust policy enforcement gate +1** (`skills/okf-lint`, `-policy-no-self-sign`)
- 계층이 다르므로 **합산하지 않는다**

asset 판정: `okf-go` DROP-AS-IS · `okf-lint` KEEP(기준 한정) · `okf-enrich` EXTEND · `okf-reader` EXTEND · `okf-producer-generator` REFERENCE · `okf-fs` REFERENCE · `okf-viz`(coverage 한정) KEEP · `okf-mcp` 보류 · connector 10종 NOT TESTED

F8 Skill policy path **REMEDIATED** / mechanical enforcement **REMEDIATED(opt-in 한정)**.
F6 **REMEDIATED(merge boundary 한정)**. F1/F2/F5 **policy requirement defined / implementation not started**. F7 미착수.

## 남은 것

**설계 판단 필요** (기계적으로 못 함)
1. `Sources` all-or-nothing — 항목 동일성 규칙 설계가 선행. `Resource`(REQUIRED) vs `ID`(optional). 잘못 고르면 provenance 가 조용히 합쳐지거나 중복된다
2. 게이트 사각지대 — B5(`Sources`) 의 fresh-non-empty 방향이 게이트에 없다. 1번 확정 후 확장
3. `UsageWindow`/`Status`/`StaleAfter` — fresh 우선이 맞는지 = ownership semantics
4. F1/F2/F5 구현 — key 이름 · schema · state vocabulary · 분자/분모 · policy metric 위치
5. F7 — §2 don't-clobber 와 재진입 규칙이 서로 막는다. 어느 쪽을 꺾을지 결정

**검증 부채** (재는 것): connector 10종(쓸 것만) · `okf-mcp` runtime · deterministic relationship handling(한 번도 못 쟀다) · `okf-viz` 나머지 · `okf-enrich` 지침 본문 적대검증

**미결정**: upstream PR(P1·P2만) · 모듈 경로 변경 · spec 드리프트 감시 · **`-policy-no-self-sign` CI 활성화(안 켜면 게이트가 죽은 코드다)**

## 이 작업의 규칙 — 반드시 유지

1. **측정하지 않은 것을 주장하지 않는다.** 하루 종일 이것 때문에 결론이 세 번 뒤집혔다. 추정은 추정이라고 쓴다
2. **SPEC 위반과 우리 정책 위반을 섞지 않는다.** `generated.by == verified.by` 는 SPEC 위반이 아니라 우리 정책 위반이다. `okf-lint` 에서도 `Conformance` 와 `PolicyFinding` 이 타입 단위로 분리되어 있다
3. **판정은 2층으로 낸다.** policy path 와 mechanical enforcement 를 따로 쓴다. 하나 닫혔다고 `CLOSED` 라고 쓰지 않는다
4. **upstream 계약 위반과 우리 요구 대비 결손을 분리 기록한다.** "상류가 약속하지 않았다"는 우리가 안전하다는 뜻이 아니다
5. **범위부터 잰다.** 오늘 패턴이 일관됐다 — 범위를 먼저 재면 작업이 작아졌고, 안 재면 뒤집혔다
6. **게이트를 깨면 fork 의 존재 이유가 회귀한 것이다.** 상류 병합 후 반드시 `go test -run TestBoundary`
7. 코드 수정 전에 **baseline FAIL 을 먼저 재현**한다. 통과만 하고 실패하지 않는 테스트는 게이트가 아니다

## 리포트 읽는 법

`OKF-RE-VALIDATION.md` 는 §0–§9 가 1차 검증, **부록 A–I 가 그 위에 얹은 재검토다. 부록이 본문을 갱신한 경우 부록이 최신이다.**

바쁘면 **§9 + 부록 A.4**(판정) → **부록 I.8**(남은 것) 만 봐도 된다.

`DERIVED-REQUIREMENTS.md` 는 리포트가 아니라 **미확정 요구사항 초안**이다. `REQUIRED` 로 취급하지 않는다.
