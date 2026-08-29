# OKF 공개 자산 독립 재검증 (2026-08-29)

**Private.** 내부 프로젝트 명칭(G-Core, okf-oracle)이 포함되어 있다. 공개 전환 금지.

| 파일 | 내용 |
|---|---|
| [`OKF-RE-VALIDATION.md`](OKF-RE-VALIDATION.md) | 검증 리포트 1294줄 — 증거와 asset별 판정 |
| [`DERIVED-REQUIREMENTS.md`](DERIVED-REQUIREMENTS.md) | 파생 요구사항 초안 (미확정) |
| [`OPEN-AT-WORK.md`](OPEN-AT-WORK.md) | 다른 PC에서 여는 법 |
| [`okf-go-fork.patch`](okf-go-fork.patch) | upstream 대비 okf-go diff (참고용) |

## 코드는 별도 저장소

**https://github.com/noplannomercy/okf-skills-fork** (private)

의도적으로 분리했다 — upstream(`xSAVIKx/okf-skills`)에 PR을 열 가능성이 있고, PR은 브랜치 단위라 내부 문서가 같은 저장소에 있으면 diff에 딸려 나갈 수 있다.

## 읽는 순서

`OKF-RE-VALIDATION.md` 는 §0–§9 가 1차 검증이고 **부록 A–I 가 그 위에 얹은 재검토다. 부록이 본문 판정을 갱신한 경우 부록이 최신이다.**

바쁘면: **§9 + 부록 A.4**(asset별 최종 판정) → **부록 I.8**(남은 것).

## 현재 상태 요약

| Asset | 판정 |
|---|---|
| `okf-go` | **DROP-AS-IS** (as-is 채택 불가) — 단 fork 비용은 2파일 +48/−8 로 측정됨 |
| `okf-lint` | **KEEP** (conformance+coverage 게이트 한정) |
| `okf-enrich` | **EXTEND** |
| `okf-reader` | **EXTEND** |
| `okf-producer-generator` | **REFERENCE** |
| `okf-fs` | **REFERENCE** |
| `okf-viz` (`coverage` 한정) | **KEEP** (진행률 계수기 한정) |
| `okf-mcp` | 판정 보류 (EVIDENCE INCOMPLETE) |
| connector 10종 | **NOT TESTED / OUT OF SCOPE** |

- Producer boundary gate **16/16** · Trust policy enforcement gate **+1**
- `okf-oracle` integration **HOLD-ENV** — 월요일 첫 작업
