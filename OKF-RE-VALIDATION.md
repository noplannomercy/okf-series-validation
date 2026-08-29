# OKF 공개 자산 독립 재검증 (Independent Re-validation)

> **읽는 순서.** §0–§9 가 1차 검증과 asset 별 판정이고, 부록 A–I 가 그 위에 얹은 재검토·수정·측정이다. **부록이 본문 판정을 갱신한 경우 부록이 최신이다** — 특히 §9.1 의 "결손은 국소적" 서술은 부록 A.4.2 에서 철회되었고, `okf-go` 판정은 A.4 에서 DROP-AS-IS 로, `okf-enrich` 는 부록 C.4 에서 EXTEND 로 확정되었다.
>
> 미해결 항목은 열려 있다(부록 I.8). 이 문서는 계속 append 된다.

## 0. 검증 조건

| 항목 | 값 |
|---|---|
| 수행일 | 2026-08-29 |
| 검증 대상 1 | `xSAVIKx/okf-skills` @ `9740e89309b0a10b0e847ac533cc33a7b4513f5c` (master, 2026-08-07T11:31:05+02:00, "chore: release master (#45)") |
| 검증 대상 2 (spec 정본) | `GoogleCloudPlatform/open-knowledge-format` @ `ad30107c31c06aec8a7d5636e0d1058118604e6f` (main, 2026-08-21T13:08:36-07:00) |
| spec 정본 | 위 repo의 `SPEC.md` — 문서 자체가 **Version 0.2** 명시 |
| 툴체인 | Go 1.26.7 windows/amd64 (포터블, sha256 `f4f534a4…abb11` = go.dev 공표값 일치) |
| 격리 | GOROOT/GOPATH/GOCACHE/GOMODCACHE 전부 scratch 내부. 시스템 PATH·레지스트리 무변경 |
| 기존 프로젝트 | `C:\workspace\prj20060203\okf_series` 및 okf-oracle — 읽지 않음, 수정하지 않음, evidence로 사용하지 않음 |
| 이전 판정 | 참조하지 않음 (어제 수행 주체·결과·실패 원인 일절 미참조·미추정) |

### 검증 원칙 적용
- upstream 계약 위반과 **우리 요구 대비 결손**을 항상 분리 기록한다.
- 결함 1건을 repo 전체로 확대하지 않는다.
- 예상 밖 결과는 즉시 결론화하지 않고 최소 재현으로 계층을 분리한다.
- asset 책임 범위를 벗어나는 요구는 FAIL이 아니라 **N/A**로 기록한다.

---

## 1. Repository Inventory (직접 확인)

`okf-skills`는 `go.work` 기반 Go monorepo다. 제공받은 asset 이름 목록을 정답으로 전제하지 않고 checkout에서 직접 열거했다.

### 1.1 Go module (16개)
```
okf-go/                      ← 공용 라이브러리
okf-mcp/                     ← MCP 서버
skills/okf-bigquery/  okf-csv/  okf-fs/  okf-git/  okf-graphql/
skills/okf-lint/  okf-mongodb/  okf-mysql/  okf-openapi/
skills/okf-postgresql/  okf-sqlite/  okf-viz/
tests/                       ← 별도 통합테스트 모듈
tools/scaffold/
```

### 1.2 instruction-only Skill (Go module 아님, 3개)
```
skills/okf-enrich/               (CHANGELOG.md, eval/, SKILL.md)
skills/okf-producer-generator/   (CHANGELOG.md, okf-go-api.md, okf-SPEC.md, SKILL.md)
skills/okf-reader/               (CHANGELOG.md, SKILL.md)
```
→ `go.work`의 `use` 목록에서도 이 3개만 제외되어 있다. **Go runtime 검증 대상이 아니며 별도 Skill 기준으로 평가한다.**

### 1.3 의존 구조 (핵심)
- **Go skill 12/12 전부가 `github.com/xSAVIKx/okf-skills/okf-go`에 의존한다.** `okf-mcp`도 의존.
- `tests/`, `tools/scaffold`는 okf-go 미의존.
- ⇒ **connector에서 관측되는 read/write/merge 현상은 okf-go 계층일 가능성이 구조적으로 높다. 계층 분리 실험이 필수다.**

### 1.4 부수 관측 (repo 위생)
`skills/okf-csv/okf-csv`, `skills/okf-lint/okf-lint`, `skills/okf-viz/okf-viz`, `skills/okf-graphql/okf-graphql` 이 **ELF 64-bit Linux 실행 바이너리로 커밋되어 있다.** 기능 결함은 아니나 공급망/위생 관점 기록 대상.

### 1.5 검증 대상 선정 및 근거
1. **`okf-go` 우선** — 12개 skill의 공통 의존이라, 여기 결과가 나머지 판정의 전제가 된다.
2. `okf-lint` (validator), `okf-fs`/`okf-sqlite` (계층 분리 재현용 대표 connector), `okf-mcp`.
3. instruction-only 3종은 Skill 기준으로 별도 평가.
4. DB/API connector 다수(bigquery, mysql, postgresql, mongodb, graphql, openapi, viz)는 외부 서비스 의존이 크고 G-Core Knowledge Baseline 직접 관련성이 낮아 현재 범위 밖 → `NOT TESTED / OUT OF CURRENT SCOPE`.

---

## 2. asset: `okf-go` (공용 라이브러리)

| 항목 | 내용 |
|---|---|
| locator | `okf-skills/okf-go` @ `9740e89`, 태그 `okf-go/v0.9.0` |
| 성격 | 12개 Go skill 전부 + okf-mcp의 공통 의존 라이브러리 |
| upstream tests | `go test ./...` → **ok  github.com/xSAVIKx/okf-skills/okf-go  0.927s (PASS, exit 0)** |

### 2.1 E1 — read→write 왕복 충실도 (unknown key)

**실험**: `ReadConceptDoc` → `WriteConceptDoc` 왕복 후 diff.
**locator**: `okf.go` — `ReadConceptDoc` / `WriteConceptDoc`, `type Frontmatter struct`

1차 시도는 내 입력이 spec 비적합(`sources`에 `uri`/`kind` 사용 — spec §5.1은 `resource`가 REQUIRED)이어서 **판정에서 폐기**하고, spec 적합 입력으로 재실행했다.

**E1b observed result (spec 적합 입력)**
- spec 정의 필드 전부 **byte-identical** 보존: `type/title/description/resource/tags/generated/verified/sources(id,resource,title,author,usage_count,last_modified)/usage_window/status/stale_after/content_hash/enriched_against`
- 한글·emoji·따옴표 등 **인코딩 충실도 PASS**
- 본문 내 `---` 수평선 포함 body 보존 PASS
- **소실된 것**: top-level 확장 키 `gcore_evidence_id`, `gcore_trust_note`, `custom_block`(중첩 포함) / `sources` 항목 내부 확장 키 `gcore_evidence_id`

**원인 (source 확인)**: `Frontmatter`는 catch-all(`yaml:",inline"` map 등)이 없는 닫힌 struct이고, `ReadConceptDoc`은 non-strict `yaml.Unmarshal`, `WriteConceptDoc`은 struct만 재marshal 한다.

**upstream 계약 대비 판정**
- SPEC §4.1: *"Consumers **SHOULD preserve unknown keys when round-tripping** and **MUST NOT reject** documents with unrecognized fields."*
- `MUST NOT reject` → **충족**
- `SHOULD preserve unknown keys` → **위반**

**우리 요구 대비**: 경계 항목 *unknown/custom metadata preservation* → **FAIL**

### 2.2 E2 — incremental regeneration 보존 (`MergeConcept`)

**locator**: `hash.go` — `MergeConcept`, `preserveExtraSections`, `unionTags`
**upstream 주장 계약** (doc comment): 구조 무변경 시 기존 문서를 그대로 반환하고 쓰기 자체를 생략, 구조 변경 시 *"agent-owned content: the existing Description, the union of Tags, and any body sections the connector did not regenerate"* 를 이월.

**observed result**

| 필드 | CASE A (구조 변경) | CASE B (구조 동일) |
|---|---|---|
| `verified` (human 검증 기록) | **소실** | 보존 |
| `sources` credibility (`id`,`title`,`author`,`usage_count`) | **소실** — `resource:` 만 잔존 | 보존 |
| `usage_window` | **소실** | 보존 |
| `status` | **소실** | 보존 |
| `stale_after` | **소실** | 보존 |
| `description` (enrichment) | 보존 | 보존 |
| `tags` | 합집합 보존 | 보존 |
| `enriched_against` | 보존 | 보존 |
| 에이전트 추가 본문 섹션 | 보존 | 보존 |
| `content_hash` | 신규 스탬프 | 유지 |
| 반환 `changed` | `true` | `false` (쓰기 생략) |

**계층 구분**
- **upstream 계약 위반 아님** — upstream은 description/tags/본문 섹션만 이월한다고 명시했고, 실제로 그대로 동작한다.
- **spec 모델과는 충돌** — SPEC §5.2: *"`verified` is independent of `generated.at`: **content can change without re-confirmation**."* spec은 내용 변경과 검증 기록을 독립 축으로 규정하는데, `MergeConcept`은 구조 변경 시 `verified`를 폐기해 두 축을 결합시킨다. 명시적 MUST 위반은 아니다.
- **우리 요구 대비**: *trust/verification state preservation* → **FAIL** / *provenance preservation* → **PARTIAL FAIL**(resource만 잔존) / *enrichment preservation* → **PASS** / *incremental regeneration semantics* → 구조 무변경 경로는 **PASS**(무변경 시 쓰기 생략으로 바이트·mtime 보존)

### 2.3 경계 항목 현황 (okf-go, 진행 중)

| 경계 항목 | 현재 상태 |
|---|---|
| encoding fidelity | **PASS** (E1b) |
| source evidence fidelity (spec 정의 필드) | **PASS** (E1b) |
| unknown/custom metadata preservation | **FAIL** (E1b) |
| enrichment preservation | **PASS** (E2) |
| provenance preservation | **PARTIAL FAIL** (E2 CASE A) |
| trust/verification state preservation | **FAIL** (E2 CASE A) |
| incremental regeneration semantics | **PARTIAL PASS** (무변경 경로 PASS / 변경 경로 위 결손) |
| deterministic relationship handling | 미검증 |
| snapshot/reproducibility semantics | 미검증 |
| stale/delete handling | 미검증 |
| malformed/missing input failure semantics | 미검증 |
| partial failure/recovery behavior | 미검증 |
| generated OKF v0.2 compatibility | 미검증 |

**판정: 보류 (검증 진행 중)** — 남은 경계 항목 확인 후 확정한다.

---

## 3. E3 — 계층 분리 실험 (connector vs 공용 라이브러리)

**목적**: E1b/E2에서 관측한 결손이 connector 결함인지 okf-go 계층 귀속인지 분리.
**대상**: `skills/okf-fs` (외부 서비스 의존 없는 대표 connector)
**upstream tests**: `go test ./...` → **ok  skills/okf-fs  0.551s (PASS)**

**절차**
1. `okf-fs produce --dir src --out bundle` 로 번들 생성
2. 생성된 `bundle/orders.csv.md` 에 사람 검증·provenance·lifecycle·확장 키·enriched description·에이전트 본문 섹션 주입
3. 원본 `src/orders.csv` 구조 변경(컬럼 추가) + `src/README.md` 삭제 + `src/new.txt` 추가
4. 동일 out 경로로 재 `produce`

**observed result (재생성 후 `bundle/orders.csv.md`)**

| 주입 항목 | 결과 |
|---|---|
| `description` (사람이 다듬음) | **보존** |
| `tags` | 합집합 **보존** |
| 본문 `## 운영 메모` | **보존** |
| `verified` (human:vavagirls) | **소실** |
| `sources` (id/title/author/usage_count) | **소실** |
| `status: stable` | **소실** |
| `stale_after` | **소실** |
| `gcore_evidence_id`, `gcore_trust_note` | **소실** |

**계층 귀속 결론**: 라이브러리 단독 실험(E2)과 connector end-to-end(E3)의 결과가 **동일**하다.
⇒ 이 결손은 **connector 결함이 아니라 `okf-go` 공용 계층에 귀속**된다. connector는 정상적으로 okf-go 계약을 사용하고 있을 뿐이다.
⇒ 역으로, 이 결손을 이유로 개별 connector를 감점하지 않는다.

### 3.1 stale / delete handling (E3 부수 관측)

원본 `src/README.md` 삭제 후 재 produce:
- `bundle/README.md.md` **잔존** (orphan concept doc, 삭제되지 않음, `status: deprecated` 표시 없음)
- `index.md`는 해당 링크를 정상 제거 ⇒ **index에서 참조되지 않는 유령 문서 발생**
- `log.md`에 Deletion 항목 없음 (기존 Creation 기록만 잔존)

**계층 구분**: okf-fs `SKILL.md`의 `produce` 계약은 "디렉토리를 순회해 번들을 출력"이며 삭제 정리(prune)를 주장하지 않는다 ⇒ **upstream 계약 위반 아님**. 우리 요구 경계 항목 *stale/delete handling* 대비 → **FAIL**.

---

## 4. asset: `okf-lint` (validator)

| 항목 | 내용 |
|---|---|
| locator | `okf-skills/skills/okf-lint` @ `9740e89` |
| upstream tests | `go test ./...` → **ok  skills/okf-lint  0.500s (PASS)** |
| 의존 | okf-go (모듈 프록시 경유 `okf-go v0.9.0` = HEAD 태그와 동일) |

### E4 — malformed / missing input failure semantics

의도적으로 훼손한 번들 투입:

| 케이스 | 결과 |
|---|---|
| frontmatter 전무 (`no_fm.md`) | **탐지** `[concept-unparseable] … missing frontmatter boundaries` |
| YAML 파싱 실패 (`bad_yaml.md`) | **탐지** `[concept-unparseable] … yaml: line 2: found unexpected end of stream` |
| `type: ""` (빈 타입) | **탐지** `[concept-missing-type]` |
| `type: TotallyMadeUpType` (미지 타입) | **거부하지 않음** → SPEC §11 *"MUST NOT reject … Unknown `type` values"* **준수** |

**exit code (정확 측정)**
- 훼손 번들 → **exit 1**
- 정상 번들 → **exit 0**

> 측정 주의: 1차 측정에서 exit 0으로 관측되었으나 이는 `| head` 파이프의 종료코드를 읽은 **측정 오류**였다. 파이프 없이 재측정하여 정정했다. 판정은 재측정값 기준.

⇒ 문서화된 *"exit 1 on violations"* 계약이 실제로 성립. **deterministic fail-closed 성립.**

**책임 범위 밖 (N/A)**: E3에서 만든 stale orphan 문서(`README.md.md`)가 포함된 번들에 대해 `conformance: 0 / exit 0`을 반환했다. validator는 원본 소스에 접근하지 않으므로 소스 삭제 탐지는 책임 범위 밖 ⇒ **FAIL 아님, N/A**.

---

## 5. E5 — snapshot / reproducibility semantics

| 실험 | 결과 |
|---|---|
| 동일 소스로 2회 fresh produce (같은 초) | **byte-identical** |
| 동일 소스로 3초 간격 fresh produce | `generated.at`, `timestamp` **만** 상이. 그 외 전부 동일 |
| `content_hash` (구조 해시) | 3회 실행 전부 **동일** — 결정적 |
| 기존 번들에 재 produce (구조 무변경) | `Unchanged, preserved: …` 출력, **쓰기 생략** ⇒ 바이트/mtime churn 없음 |

> 측정 주의: 최초 2회가 동일했던 것은 같은 초 실행에 따른 우연이었다. 시간차 재현으로 분리 확인함.

**판정**: 경계 항목 *snapshot/reproducibility semantics* → **PARTIAL PASS**
- 구조 해시 결정성 **PASS**, 증분 경로 무churn **PASS**
- 벽시계 시각이 산출물에 박히므로 **from-scratch 바이트 재현은 불성립**

---

## 6. 생성물의 OKF v0.2 적합성 (부수 관측)

`okf-fs` 생성 번들 실측:
- `index.md` 에 `okf_version: "0.2"` 명시
- `okf-lint` conformance 위반 **0건**, exit 0
- `index.md`에 `type` 없음 / `log.md`에 frontmatter 없음 → 둘 다 예약 파일(§8, §9)이며 lint도 concept 집계에서 제외 ⇒ 위반 아님
- `timestamp`(v0.1 legacy)를 `generated`와 **병기** — spec상 확장 키로 허용 범위, 거부 사유 아님
- 관측된 사소한 사항: 자산 링크 표시 텍스트에 Windows 구분자 노출(`[docs
ote.txt](docs/note.txt.md)`). 링크 대상은 정상(broken links 0)

---

## 7. E6 — `stale_after` 파싱: 정본 spec 대비 결함 (중대)

**locator**: `okf-go/okf.go` — `func (fm Frontmatter) IsStale(now time.Time) bool`
**구현**: `time.Parse("2006-01-02", fm.StaleAfter)` — 날짜 전용 레이아웃. 파싱 실패 시 `false` 반환.

**observed result** (`now = 2026-08-29T12:00:00Z`)

| `stale_after` 값 | `IsStale` | 비고 |
|---|---|---|
| `2020-01-01T00:00:00Z` (정본 spec 예시 형식, 6년 경과) | **false** | **오탐 — 낡지 않음으로 처리** |
| `2030-01-01T00:00:00Z` (정본 형식, 미래) | false | (우연히 결과만 일치) |
| `2020-01-01` (날짜 전용, 과거) | true | 정상 |
| `2030-01-01` (날짜 전용, 미래) | false | 정상 |
| `""` (부재) | false | 정상 |
| `not-a-date` (쓰레기 값) | **false** | **fail-open, 오류 미노출** |

### 7.1 근본 원인 — 상류 spec의 무버전 드리프트

| 문서 | §5.5 규정 |
|---|---|
| GCP `SPEC.md` @ `ad30107` (**정본**) | `stale_after: 2026-09-23T00:00:00Z` — *"An absolute **instant**… `now >= stale_after`"* |
| xSAVIKx 번들 `okf-SPEC.md` @ `9740e89` | `stale_after: 2026-09-23` — *"An absolute **date** (`YYYY-MM-DD`)… `today >= stale_after`"* |

두 문서 **모두 "Version 0.2"를 표방**한다. GCP repo 커밋 이력에서 원인이 확인된다:

```
3dc3029 make every timestamp an ISO 8601 datetime with an explicit offset
ad30107 Merge pull request #6 … okf-iso-datetimes   (2026-08-21)
```

⇒ 정본 v0.2가 **버전 번호를 올리지 않은 채 제자리 변경**되었고, okf-skills HEAD(2026-08-07)는 그 이전 스냅샷을 번들하고 있다.

**계층 구분 (중요)**
- **2026-08-07 시점 v0.2 대비**: okf-go는 **정합**. 구현 부실이 아니다.
- **현재 정본 v0.2 대비**: **불일치**. 정본 예시 형식이 조용히 fail-open 된다.
- **우리 요구 대비**: staleness gate를 okf-go에 위임하면 **낡은 지식이 신선한 것으로 통과**한다 → 경계 항목 *stale handling* / *partial failure semantics* **FAIL**
- 영향 범위: 공용 계층이므로 **Go skill 12개 + okf-mcp 전체에 전파**된다.

---

## 8. instruction-only Skill 평가

평가 기준: Input/Observation → Judgment → Action → Verification → Stop/Recovery + 실제 반복 구현량·판단 오류 감소 효과. **validator식 deterministic fail-closed 기준을 적용하지 않는다.**

| Skill | Input/Obs | Judgment | Action | Verification | Stop/Recovery |
|---|---|---|---|---|---|
| `okf-reader` v0.3.0 | index-first 발견, frontmatter-only 파싱, grep 라우팅 | trust tier 도출, staleness, lifecycle gating, per-claim 귀속 | 개념 라우팅, 링크 추적, `resource` 역참조 | 번들 sanity-check, attester 영수증 확인 | broken link를 오류가 아닌 "미작성 지식"으로 처리, 링크 부재 허용 |
| `okf-enrich` v0.4.0 | index-first, 스키마/프로파일/샘플/관계 근거 수집 | 무엇을 보강할지 판단, 근거 없는 주장 금지, 카디널리티 근거화 | description·관계 주석·태그 기입 | **`okf-viz coverage` 재측정 루프**, `--min` 게이트, `eval/rubric.md` 5점 척도 LLM-judge | 실질적 설명 덮어쓰기 금지, 멱등·재실행 안전, 모호하면 불확실성 명시 |
| `okf-producer-generator` v0.4.0 | 소스→개념 매핑 | 5원칙(produce 내 LLM 금지, okf-go 단일 타입원, schema 계약, 3서브커맨드, pure-Go) + sync 3패턴 결정표 | 스캐폴딩·구현·등록 체크리스트 | **"Verify before you call it done" 5단계** (build/produce/ingest/schema/validate) | "Common mistakes" 복구표, "새 메커니즘 발명 금지, validate-only로 기본화" |

**교차 정합성 확인 (직접 수행)**
- `okf-reader`의 trust tier 규칙 ↔ `okf-go.GetTrustTier` 구현 ↔ SPEC §5.3 → **3자 완전 일치**
- `okf-reader`의 staleness 규칙(`YYYY-MM-DD`) ↔ okf-go 구현 → 일치. 단 **정본 spec과는 불일치** (§7)
- `okf-enrich` §4c *"Preserve existing verifications — append … without dropping existing entries"* ↔ `MergeConcept` 실제 동작 → **충돌**. 스킬 지침은 spec에 부합하나, 구조 변경 재생성 시 라이브러리가 `verified`를 폐기한다(§2.2, §3). **결손은 라이브러리 계층이며 스킬의 결함이 아니다.**

---

## 9. 최종 판정

| Asset | 판정 | 근거 요약 |
|---|---|---|
| **`okf-go`** (공용 라이브러리) | **EXTEND** | upstream test PASS. spec 정의 필드 왕복 **byte-identical**, 인코딩 충실도·구조해시 결정성·증분 무churn 확보. 그러나 우리 경계 항목 3개가 **국소적으로** 깨진다 — ①확장 키 소실(§2.1) ②구조 변경 시 `verified`/`sources` 상세/`status`/`stale_after` 폐기(§2.2) ③`stale_after` 정본 형식 fail-open(§7). 셋 다 **특정 함수에 한정**되고 아키텍처 결함이 아니다. 핵심은 재사용 가능, 제한된 수정 필요. |
| **`okf-lint`** (validator) | **KEEP** | upstream test PASS. malformed/무frontmatter/빈 type 전부 탐지, **위반 시 exit 1 / 정상 시 exit 0** 확인. 미지 `type`은 거부하지 않아 SPEC §11 준수. 결정적 fail-closed 성립 ⇒ 무수정 CI 게이트로 사용 가능. *한계(결함 아님)*: 소스 접근이 없어 stale/삭제 탐지는 **N/A**, trust/provenance 의미 검증은 계약 범위 밖. |
| **`okf-reader`** (instruction-only) | **EXTEND** | 절차 5요소 완비, trust tier 규칙이 구현·spec과 3자 일치. 다만 `stale_after`를 `YYYY-MM-DD`로 기술해 **정본 v0.2와 드리프트**(§7) ⇒ 해당 규칙 한 항목의 제한적 갱신 필요. |
| **`okf-enrich`** (instruction-only) | **KEEP** | 절차 5요소 완비 + **재측정 루프**(`okf-viz coverage`)와 **자체 채점 루브릭**(5차원×1–5점) 보유. 근거기반 서술·멱등성·검증기록 append 규칙이 우리 요구와 정합. 반복 판단 오류를 줄이는 procedural value 높음. as-is 사용 가능. |
| **`okf-producer-generator`** (instruction-only) | **EXTEND** | 절차 5요소 완비, 비자명한 결정(produce 내 LLM 금지 / 비밀값 argv 금지·Env 바인딩 / `Resource`에서 자격증명 제거 / pure-Go) 을 명문화 — 재구현량 절감 효과 큼. 다만 **드리프트된 `okf-SPEC.md` 스냅샷을 동봉**하고 okf-go를 단일 타입원으로 강제하므로, 그대로 따르면 §2.1·§7 결손이 신규 producer에 그대로 복제된다 ⇒ 제한적 갱신 필요. |
| **`okf-fs`** (connector) | **REFERENCE** | upstream test PASS, produce/ingest·증분 무churn 정상. **자체 결함 없음** — 관측된 보존 결손은 전부 okf-go 계층 귀속(§3). 다만 ①대상이 파일시스템 트리로 한정되어 G-Core Knowledge Baseline의 운영 자산으로 직접 쓰기 어렵고 ②삭제 정리 미수행으로 orphan 문서 잔존(§3.1). ⇒ producer 구현 패턴의 **참조 자산**으로 유효. |
| **`okf-mcp`** | **KEEP (증거 불완전)** | upstream test PASS, `tests/` 통합 PASS. 문서 변환을 하지 않는 발견/노출 계층이라 13개 경계 항목 대부분이 **N/A**. **단, 본 검증에서 독립 runtime 실험을 수행하지 않았다.** 판정을 확정하려면 추가 검증 필요 — 현재는 잠정. |
| `okf-bigquery`, `okf-csv`, `okf-git`, `okf-graphql`, `okf-mongodb`, `okf-mysql`, `okf-openapi`, `okf-postgresql`, `okf-sqlite`, `okf-viz` | **NOT TESTED / OUT OF CURRENT SCOPE** | 인벤토리·의존관계만 확정(전부 okf-go 의존). 외부 서비스·자격증명 의존이 크고 현재 범위와 직접 관련성이 낮아 검증 대상에서 제외. **결함 판정이 아니다.** `okf-viz`는 `okf-enrich`가 coverage 재측정에 참조하므로 차기 검증 1순위. |

### 9.1 판정 요지

- **"공개 자산 전부 사용 불가"는 이번 증거로 지지되지 않는다.** 검증한 5개 모듈의 upstream 테스트가 전부 PASS했고, spec 정의 필드는 무손실 왕복하며, validator는 문서대로 fail-closed로 동작한다.
- **결손은 세 지점에 집중된다** — 확장 키 보존, 구조 변경 시 trust/provenance 이월, `stale_after` 형식. 셋 다 `okf-go` 단일 계층이며 국소적이다.
- **connector를 감점할 근거는 발견되지 않았다.** 계층 분리 실험(§3)으로 귀속을 확정했다.
- **instruction-only Skill 3종의 procedural value는 실질적이다.** 특히 `okf-enrich`의 재측정 루프·채점 루브릭과 `okf-producer-generator`의 완료 검증 5단계는 반복 판단 오류를 직접 줄인다.

### 9.2 본 검증의 한계 (명시)

- connector 10종 미검증(위 표). 이들 고유 결함 유무는 **알 수 없음** — 무결하다는 뜻이 아니다.
- `okf-mcp`는 upstream test·정적 확인까지만 수행.
- `deterministic relationship handling`은 `okf-fs`가 관계 섹션을 생성하지 않아 본 검증에서 실측하지 못했다(SQL/git connector 대상 후속 필요) ⇒ 현재 **미검증**.
- 실험은 Windows/amd64 단일 플랫폼에서 수행했다.

> 원칙 9·10에 따라 기존 구현 수정, 수정 방법 제안, 대체 architecture 제안은 이 문서에 포함하지 않는다. evidence와 판정까지만 기록한다.


---
---

# 부록 A. Adversarial Adequacy Review (자체 감사 및 재판정)

> §1–§9의 evidence는 **폐기하지 않는다.** 아래는 그 위에 얹는 적대적 재검토이며, 새 evidence로 판정이 바뀐 항목만 재판정한다.

## A.0 자체 편향 감사 — 무엇이 편향이었고 무엇이 아니었나

**편향이 아니었던 것 (기록으로 반증됨)**
§1–§9에서 실제로 투입한 negative/adversarial fixture: frontmatter 전무 · YAML 파싱 실패 · `type: ""` · 미지 `type` · `stale_after` 쓰레기 값 · 정본 instant 형식 · top-level 및 중첩 확장 키 · 사람 검증 주입 후 구조 변경 · 원본 파일 삭제 · 비ASCII/emoji · 시간차 결정성. 또한 자기 정정을 **양방향으로** 수행했다 — 자산에 불리한 관측 1건(비적합 입력으로 인한 `sources` 파괴 오판)과 유리한 관측 1건(파이프 종료코드 오독으로 인한 fail-open 오판)을 모두 폐기했다.

**실제 편향 (인정)**
1. `okf-go`를 EXTEND로 판정할 때 **"결함이 특정 함수에 한정된다"는 코드 수정량 논리**를 사용했다. 운영 심각도·탐지 가능성 기준이 아니다.
2. *"upstream 계약 위반은 아니다"* 를 완화 요인처럼 취급했다. **채택 판정에서 이 사실은 무관하다** — 상류가 약속하지 않았다는 것은 우리가 안전하다는 뜻이 아니다.
3. 실험이 `Read/Write/Merge/IsStale`에 집중되어, **`ConceptStructuralHash`의 해시 범위**와 **`verified` 역직렬화 예외 경로**라는 두 개의 더 위험한 경로를 건드리지 않았다.
4. instruction-only Skill 3종(`okf-enrich`, `okf-producer-generator`, `okf-reader`)에 대해 **적대적 검증을 전혀 수행하지 않았다.**

⇒ 3번 항목을 겨냥해 추가 fixture를 설계·실행했고, 그 결과가 아래다.

---

## A.1 신규 발견 — A1: 구조 해시가 frontmatter를 무시한다 (**silent identity corruption**)

**locator**: `okf-go/hash.go` — `ConceptStructuralHash(doc ConceptDoc)` → `sha256(normalizeForHash(doc.Body))`
**사실**: 해시 입력은 **`doc.Body` 뿐**이다. `resource` / `type` / `title` / `tags` 등 frontmatter 전체가 해시 범위 밖이다.

**실험**: body는 동일하고 자산 신원만 바뀐 fresh 문서를 `MergeConcept`에 투입

| 항목 | 값 |
|---|---|
| `hash(existing)` | `8a0fd734…96ac` |
| `hash(fresh)` | `8a0fd734…96ac` — **동일** |
| 반환 `changed` | **`false`** → 호출자가 **쓰기를 생략** |
| `merged.type` | `"SQLite Table"` (fresh는 `"PostgreSQL Table"`) |
| `merged.title` | `"orders"` (fresh는 `"orders_v2"`) |
| `merged.resource` | **`"sqlite:///PROD-A.db#orders"`** (fresh는 `postgres://PROD-B/orders_v2`) |

**failure mode**: 기저 자산의 신원이 바뀌어도 문서는 **옛 `resource`를 계속 가리킨다.** 쓰기가 생략되므로 `log.md` 기록도, timestamp 변화도, git diff도 발생하지 않는다. 그리고 그 문서에 붙어 있던 **`verified: human:…` 도장이 잘못된 신원 위에 그대로 유지된다.**

- severity: **치명** (knowledge correctness + evidence provenance + trust state 동시 훼손)
- detectability: **없음** — 산출물 변화가 0이다. 아래 A.3에서 validator도 통과함을 실측 확인
- blast radius: 공용 계층 ⇒ **Go connector 12개 전부 + `okf-mcp`**

## A.2 신규 발견 — A2: `verified` 이형(異形) 입력의 silent 소멸

**locator**: `okf-go/okf.go` — `func (vl *VerifiedList) UnmarshalYAML(node *yaml.Node) error` — Sequence/Mapping 이외 노드는 `return nil` (오류 아님)

| 입력 | `err` | 항목 수 | trust tier |
|---|---|---|---|
| `verified: {by: human:x, at: …}` (spec 규정 형태) | nil | 1 | `human-reviewed` |
| `verified: human:vavagirls` (스칼라) | **nil** | **0** | **`unverified`** |
| `verified: null` | nil | 0 | `unverified` |
| `verified: 12345` | **nil** | **0** | **`unverified`** |
| `verified: true` | **nil** | **0** | **`unverified`** |

**failure mode**: 형태가 어긋난 신뢰 기록이 **오류 없이 사라진다.** 게이팅 방향 자체는 안전측(낮은 신뢰)이나, 그 상태로 `WriteConceptDoc`을 거치면 `omitempty`로 **디스크에서 검증 기록이 영구 삭제**된다.
- severity: 높음 (evidence 파괴) / detectability: 없음 (오류·경고 없음) / blast radius: 공용 계층 전체

## A.2b 부수 관측 (경미)

- **CRLF**: CRLF 파일은 정상 파싱되나 body가 `
`을 유지한 채 frontmatter는 `
`으로 재작성되어 **혼합 개행 파일**이 된다. `normalizeForHash`가 개행을 정규화하므로 해시는 안정. → 진단 잡음, 훼손 아님
- **body 선두 `---`**: 정상 처리(§E1b 재확인)

---

## A.3 탐지 가능성 실측 — validator는 이 훼손들을 잡지 못한다

A1/A2 시나리오를 담은 번들을 `okf-lint`에 투입:

```
bundle is 100.0% enriched (placeholders: 0, broken links: 0, orphans: 3, conformance: 0)
- concepts missing type: 0
EXIT = 0
```

번들의 실제 내용: ①`verified`가 스칼라라 신뢰 기록이 무효화된 문서 ②정본 v0.2 기준 **6년 낡은** `stale_after: 2020-01-01T00:00:00Z` 문서 ③사람 검증 도장이 붙은 채 신원이 어긋난 문서. `-strict`도 orphan(교차링크 지표)만 실패시킬 뿐 위 셋 중 어느 것도 잡지 못한다.

⇒ **제공된 도구 체인 안에 이 실패 모드들의 탐지 경로가 존재하지 않는다.**
⇒ 단, 이것을 `okf-lint`의 결함으로 계상하지 않는다. 그 계약은 *conformance + coverage*이고 SPEC §11의 conformance 정의를 정확히 이행한다. **책임 범위 밖 = N/A.** 다만 아래 재판정에서 채택 기준(acceptance criterion)을 명시한다.

---

## A.4 재판정

| Asset | 기존 판정 | 채택 기준 (acceptance criterion) | silent failure mode | severity | detectability | blast radius | adversarial coverage | 무수정 채택 | **재판정** |
|---|---|---|---|---|---|---|---|---|---|
| **`okf-go`** | EXTEND | G-Core Knowledge Baseline의 **evidence/trust 보존 계층**으로 채택 가능한가 | ①확장 키 소실 ②구조 변경 시 `verified`/provenance/lifecycle 폐기 ③정본 `stale_after` fail-open ④**해시 범위 밖 신원 변경 시 쓰기 생략(A1)** ⑤**이형 `verified` 무오류 소멸(A2)** | **치명** — 5개 중 4개가 knowledge correctness / evidence provenance / trust state를 직접 훼손 | **없음** — 전부 무오류·무경고, A1은 산출물 변화 0, validator 통과(A.3) | **12 connector + okf-mcp** (공용 단일 경로) | 충분 (신규 fixture 2종 추가) | **불가** | **DROP-AS-IS** |
| **`okf-lint`** | KEEP | **conformance + coverage CI 게이트**로 채택 가능한가 | 없음 (계약 범위 내) | — | 계약 범위 내에서는 결정적 fail-closed(exit 1) 실측 | 단독 바이너리 | 충분 | 가능 | **KEEP (기준 한정)** — *trust/evidence 무결성 게이트로는 사용 불가. A.3 참조* |
| **`okf-reader`** | EXTEND | 소비자측 **판독 절차 지침**으로 채택 가능한가 | `stale_after` 형식 기술이 정본과 드리프트 | 중 | 중 (형식 불일치는 대조로 발견 가능) | 지침 채택 범위 | **불충분** — 적대적 검증 미수행 | 불가(1개 규칙) | **EXTEND (유지)** |
| **`okf-enrich`** | KEEP | **보강 절차 지침**으로 채택 가능한가 | §4c "검증 기록 보존" 지침이 라이브러리 실동작과 충돌 — **지침의 결함 아님** | 저(지침 자체) | — | 지침 채택 범위 | **불충분** — 적대적 검증 미수행, 의존 도구 `okf-viz` 미검증 | 가능(단 아래 조건) | **KEEP (증거 불충분 표기)** |
| **`okf-producer-generator`** | EXTEND | **신규 producer 작성 운영 지침**으로 채택 가능한가 | 드리프트된 `okf-SPEC.md` 동봉 + "okf-go를 단일 타입원으로" 원칙이 A.1/A.2 설계를 신규 producer에 **복제** | 높음 (결함 전파) | 낮음 | 이 지침으로 만든 모든 producer | **불충분** | 불가 | **REFERENCE** (EXTEND→하향) |
| **`okf-fs`** | REFERENCE | — | 자체 결함 없음. A.1/A.2를 상속 | — | — | — | 충분 | — | **REFERENCE (유지)** |
| **`okf-mcp`** | KEEP(증거 불완전) | — | okf-go 의존이므로 A.1/A.2 상속 | — | — | — | **불충분** | 불가 | **판정 보류 (EVIDENCE INCOMPLETE)** |
| 미검증 connector 10종 | NOT TESTED | — | — | — | — | — | 없음 | — | **NOT TESTED (유지)** |

### A.4.1 `okf-go` 재판정 근거 — 왜 EXTEND가 아닌가

사용자가 정의한 라벨을 그대로 적용한다.
- **EXTEND** = *"핵심은 재사용 가능하지만 제한된 수정이 필요"*
- **DROP-AS-IS** = *"현재 목적에서는 수정 없이 사용할 수 없다"* (품질 폄하 아님)

우리 목적에서의 **핵심**은 "OKF 파일을 만들 수 있는가"가 아니라 **증분 재생성을 거치며 evidence·provenance·trust를 보존하는가** 이다. 그런데 훼손 경로 3개(A.1 해시 범위, §2.2 merge 이월 집합, §2.1 확장 키 미보존)는 각각 upstream이 **의도적으로 설계·문서화한 결정**이다 — 버그가 아니라 계약이다. 세 계약을 동시에 바꾸는 것은 라이브러리의 정체성을 바꾸는 일이지 *"제한된 수정"* 이 아니다. 따라서 **핵심이 재사용 가능하다는 EXTEND의 전제가 성립하지 않는다.**

동시에 다음은 실측으로 **건재함이 확인**되었고 재판정에서 부정되지 않는다: spec 정의 필드의 byte-identical 왕복, 인코딩 충실도, 구조 해시의 실행 간 결정성, 증분 무변경 경로의 무churn, `GetTrustTier`의 spec 3자 정합, `SkillSchema`/섹션 헬퍼, upstream 테스트 전건 PASS. ⇒ **DROP-AS-IS는 "쓸모없다"가 아니라 "이 역할에 무수정 채택 불가"이며, 타입 모델·직렬화·구조 해시 개념은 REFERENCE 등급으로 유효하다.**

### A.4.2 §9.1 결론의 정정

§9.1의 *"결손은 세 지점에 집중되며 국소적이다"* 는 **철회한다.** 코드 수정량 기준이었고, A.1 발견 이전이었다. 정정된 판단: 결손은 5개이며, 그중 A.1은 **산출물에 아무 변화도 남기지 않는** 훼손이라 국소성 논의 자체가 성립하지 않는다.

§9.1의 나머지 — *"공개 자산 전부 사용 불가는 이번 증거로 지지되지 않는다"*, *"connector를 감점할 근거는 없다"* — 는 **유지한다.** upstream 테스트 전건 PASS와 계층 귀속 실험(§3)은 A.1/A.2로 반증되지 않는다. `okf-lint`은 자기 계약 안에서 결정적 fail-closed로 동작함이 실측되었다.

### A.4.3 남은 불충분 (다음 검증 대상)

- `okf-enrich` / `okf-producer-generator` / `okf-reader`: 적대적 검증 **미수행**. 위 판정은 문서 독해와 교차 대조에 근거하며, 실행 근거가 없다.
- `okf-viz`: `okf-enrich`의 재측정 루프가 의존하는데 **미검증**. coverage 지표가 신뢰 가능한지 불명.
- `okf-mcp`: runtime 검증 미수행.
- `deterministic relationship handling`: 여전히 미실측.
- connector 10종: 미검증.

> 원칙에 따라 코드 수정, 수정 방법, 대체 architecture는 이 문서에 포함하지 않는다.


---
---

# 부록 B. `okf-viz coverage` 표적 검증

> **범위 한정.** `okf-viz` 전체를 평가하지 않는다. `okf-enrich`의 Verification 단계가 의존하는 **coverage 재측정이 신뢰 가능한지**만 확인한다. 본 부록은 `okf-viz`·`okf-enrich` 두 항목만 갱신하며 다른 asset 판정은 건드리지 않는다.

## B.1 검증 대상 contract (okf-enrich 원문에서 추출)

| # | 주장 | 출처 |
|---|---|---|
| C1 | *"deterministic, no-LLM coverage report"* | `okf-enrich/SKILL.md:61` |
| C2 | 보고 항목 = 비placeholder description 비율, 주석 컬럼, broken cross-link, type 누락, orphan | 동 `:61` |
| C3 | `--json` 기계 출력 / **`--min <pct>` 로 CI 게이트** | 동 `:61` |
| C4 | *"enrich these first"* 랭킹 배출 | 동 `:153` |
| C5 | 재측정 가능 ⇒ 부분 수행이 **resumable** 상태 | 동 `:149` |
| C6 | *"counts **how much** is enriched; it **cannot judge how well**"* | 동 `:207`, `eval/README.md:8` |

**구조적 사실**: `okf-viz coverage`는 `okf.ScanBundle`을 호출한다 — **`okf-lint`이 쓰는 동일한 okf-go 함수**다 (`okf-viz/main.go:68`). 두 도구는 같은 계산기를 공유하며, coverage 지표는 okf-go 계층에 귀속된다.

**숫자의 정의 (직접 확인)**
`EnrichedPct = 100 × EnrichedConcepts / TotalConcepts` (`okf-go/lint.go:164`)
개념이 "enriched"로 계수되는 조건 = `!IsPlaceholderDescription(description)`
`IsPlaceholderDescription` = 공백이면 true, 아니면 **하드코딩된 14개 앵커 정규식** 목록과 대조 (`okf-go/placeholder.go`) — `^SQLite (table|view) .+$`, `^File .+$`, `^MongoDB collection .+$`, `^No description available$` 등.

⇒ **이 숫자는 "설명이 비어있지 않고 알려진 14개 placeholder 패턴에 걸리지 않는다"를 측정한다.** 보강의 정확성·고유성·근거성은 측정 대상이 아니다.

## B.2 runtime 실측

빌드: `okf-viz` upstream tests **PASS** (`ok skills/okf-viz 0.970s`). repo dirty=0 유지.

### 계약 내 동작 — 전부 PASS

| Fixture | 조건 | 결과 | 판정 |
|---|---|---|---|
| F1 | placeholder 2 + 실설명 2 | `50.0%`, placeholders 2, enrich-first = `c1,c2` 정확 | PASS |
| F2 | 전부 보강 | `100.0%` | PASS (전/후 변화 반영) |
| F3 | description 1건 비움 | `75.0%`, placeholders 1 | PASS (**감소 탐지**) |
| F7 | placeholder 1건 | `75.0%` | PASS |
| F8 | `--min` 게이트 | `--min 50`→exit 0 / `--min 75`(경계, 75.0% 달성)→exit 0 / `--min 90`→**exit 1** | PASS (문서대로) |
| F9 | 동일 입력 3회 (`--json`) | `pct=50, placeholders=2, total=4` 3회 동일 | PASS (**결정적**) |
| F10(b) | 존재하지 않는 경로 | `Failed to scan bundle: …` **exit 1** | PASS (fail-closed) |
| F10(c) | 빈 디렉토리 (개념 0) | `0.0%`, exit 0 / `--min 90`→**exit 1** | PASS (0/0을 100%로 처리하지 않음) |
| F10(a) | 훼손 번들 | conformance 위반 3건을 **출력에 노출** | PASS (노출) |

### 계약 경계에서 뚫린 지점

| Fixture | 조건 | 결과 | 성격 |
|---|---|---|---|
| **F4** | description 전부 쓰레기 (`asdf`,`qwer`,`TODO`,`xxx`) | **100.0% enriched**, `--min 90` **exit 0(통과)** | C6가 **명시 면책** — 계약 위반 아님 |
| **F6** | 동일 설명 4회 복제 | **100.0% enriched** | C6 면책 범위 |
| **F5** | 미등록 connector placeholder (`Redis key session:1`, `Kafka topic orders`, `Neo4j node Person`, `S3 object logs/1`) | **100.0% enriched** | **면책 밖** — 품질 판단이 아니라 **개수 세기 오류**. C6("how much")의 주장 범위 안에서 틀린다 |
| **F11** | placeholder 2건을 **손대지 않고** 보강된 개념 4개만 추가 | `50.0% → 75.0%`. `--min 70`이 **exit 1 → exit 0** 으로 뒤집힘. `placeholders: 2` 는 그대로 | 비율 지표의 구조적 성질 — 게이트가 **개념 추가로 충족 가능** |
| **F10(a)** | 훼손 번들 (frontmatter 전무 / YAML 깨짐 / 빈 type) | conformance 위반 3건을 **출력하나 게이트하지 않음**. `--min` 만이 유일한 게이트이며 그 축은 enrichment% | coverage는 conformance로 fail 하지 않는다. 그 역할은 `okf-lint lint`(exit 1, §4) |

### F5의 문서 상태 — upstream에 공정하게

`okf-producer-generator/okf-go-api.md:265-269`는 명시한다:
> `IsPlaceholderDescription(desc string) bool` … **"When you add a connector, add its fallback description's pattern here"** so the coverage report and triage stay precise. Used by `okf-viz coverage`.

⇒ F5는 **미문서화 함정이 아니라 문서화된 결합**이다.
⇒ 다만 `okf-producer-generator/SKILL.md`의 **등록 체크리스트 11개 항목에 이 단계가 없다**(`go.work`/`Makefile`/`install.sh`/`skills.sh.json`/`README`/`AGENTS`/`tests`/`go mod tidy`/`skills-ref validate` 만 존재). 운영 산출물인 체크리스트가, 검증 지표를 정직하게 유지하는 항목을 누락했다 — **skill 내부 정합성 결손**이며 §A.4에서 이미 REFERENCE로 하향한 판정과 일관된다.

## B.3 결론 — 이 숫자가 Verification 주장과 연결되는가

**coverage가 신뢰 가능하게 답하는 질문**: "설명이 아직 **비어 있거나 알려진 placeholder인** 개념이 몇 개인가."
→ 결정적이고, 전/후 변화를 반영하며, 감소를 탐지하고, 게이트 exit code가 문서대로다.

**coverage가 답하지 못하는 질문**: "그 설명이 **맞는가**"(F4) · "**서로 다른가**"(F6) · "**새 connector의 placeholder는 아닌가**"(F5) · "게이트 통과가 **실제 보강 때문인가**"(F11).

⇒ `okf-enrich`의 Verification 단계는 **완성도 카운터이지 정확성 검사가 아니다.** 이는 `okf-enrich` 자신이 C6에서 밝힌 바와 일치하므로 **skill의 부정직이 아니다.** 그러나 우리 목적(evidence/trust integrity)에서는, 규정된 Verification을 그대로 채택하면 §A.1(신원 훼손)·§A.2(신뢰 기록 소멸) 같은 silent corruption을 **하나도 탐지하지 못한다.** 실제로 §A.3에서 동일 계산기(`ScanBundle`)를 쓰는 `okf-lint`이 그 번들에 `100.0% enriched / conformance 0 / exit 0`을 준 바 있다.

## B.4 판정 갱신 (2건만)

| Asset | 기존 | 채택 기준 | adversarial coverage | 무수정 채택 | **재판정** |
|---|---|---|---|---|---|
| **`okf-viz` (`coverage` 서브커맨드 한정)** | 미검증 | **결정적 진행률·트리아지 카운터**로 채택 가능한가 | 충분 (F1–F11) | **가능 (기준 한정)** | **KEEP (기준 한정)** — 진행률/트리아지 계수기로는 결정적·fail-closed로 동작. **knowledge integrity 검증 게이트로는 사용 불가.** `okf-viz`의 나머지 서브커맨드는 **NOT TESTED** |
| **`okf-enrich`** | KEEP (증거 불충분) | **보강 절차 지침**으로 무수정 채택 가능한가 | 부분 확보 — Verification 의존 도구를 실측(B.2). 지침 본문 자체의 적대적 검증은 **여전히 미수행** | **불가** | **EXTEND** (KEEP→하향) |

### B.4.1 `okf-enrich` 하향 근거

절차 5요소(Input/Judgment/Action/Verification/Stop-Recovery)는 §8에서 확인한 대로 모두 존재하고, 근거기반 서술·멱등성·검증기록 append 규칙은 spec과 정합하며 **그 판단은 유지한다.** 하향 사유는 단 하나 — **Verification 단계 하나**다. 규정된 재측정 도구가 완성도만 세므로, 이 지침을 무수정 채택하면 우리 목적에 필요한 검증이 성립하지 않는다 ⇒ KEEP의 정의("수정 없이 채택 가능")를 충족하지 못한다.

`okf-go`(§A.4.1)와 달리 **EXTEND가 성립하는 이유**: Verification은 지침 문서 안의 **독립된 단계**이고, 다른 단계 중 coverage의 의미론에 의존하는 것이 없다. 즉 **안전한 확장 지점이 실제로 식별된다.** 반면 okf-go의 훼손 경로 3개는 upstream이 의도적으로 설계한 계약 자체였다. 두 하향의 성격이 다르다.

### B.4.2 본 부록의 한계

- `okf-viz`의 `coverage` 외 서브커맨드(렌더/링크/통합) **미검증**.
- `okf-enrich` **지침 본문**에 대한 적대적 검증(실제 보강 산출물 품질)은 여전히 미수행. B.2는 의존 도구만 검증했다.
- `ColumnsCommented*` 지표는 SQL connector 산출물이 필요해 본 fixture로 실측하지 못했다 ⇒ **미검증**.
- 다른 asset 판정은 본 부록에서 변경하지 않았다.


---
---

# 부록 C. `okf-enrich` Skill 자체 적대검증

> **방법론 선언.** `okf-enrich`는 instruction-only다. "지침을 따르면 에이전트가 환각하는가"를 실행으로 재려면 검증자 자신이 피험자가 되어야 하고, 함정을 읽은 상태에서 자기 산출물을 채점하는 것은 구조적으로 PASS 편향이며 반증 불가능하다. 따라서 본 부록은 **자가수행 채점을 하지 않는다.** 대신 ①규범 명제의 **강제력** 분류 ②지침이 의존하는 **메커니즘의 기계적 실측** ③위반이 규정된 Verification(§B 실측 완료)으로 **탐지 가능한가** 로 판정한다.

**강제력 등급**: ⓐ 도구로 기계 검사 가능 · ⓑ 산출물 대조로 사람이 검사 가능 · ⓒ **검증 훅 없이 모델 판단에만 의존** · ⓧ **규칙 자체가 부재**

## C.0 원문 확인 — 존재하지 않는 규칙

| 항목 | 결과 |
|---|---|
| 충돌하는 evidence 처리 절차 | **부재** (`SKILL.md`/`eval/` 전수 검색. `rubric.md:20`의 *"contradicts or ignores the schema"* 는 점수 1 앵커일 뿐 절차 아님) |
| 기존 추론의 철회·하향 규칙 | **부재** (`enriched_against` 재진입 메커니즘만 존재) |
| 근거 부족 시 보강 **보류/거부** 경로 | **부재** — `skip`은 오직 *"변경 없음"* 용도(`:176,180,221`). "근거가 없으니 쓰지 않는다"는 출구가 문서에 없다 |
| rubric의 불확실성 보정 / 충돌 처리 차원 | **부재** (4차원 = Grounding, Specificity, Conciseness, Surgical/idempotent) |

## C.1 의존 메커니즘 기계 실측 (판단 개입 없음)

`okf-go` @ `9740e89` 대상, 지침의 frontmatter 연산을 문자 그대로 적용.

| ID | 실측 내용 | 결과 |
|---|---|---|
| **T1** | §4c *"append to the list"* 를 3회 반복 | `verified` 항목이 **1→2→3 으로 중복 누적**. 동일 actor·동일 시각도 중복 제거되지 않음. **대조**: §4b tags는 *"union, deduplicated, sorted"* 를 명시 — 동일 문서 내 규칙 비대칭 |
| **T2** | 설명을 방금 작성한 actor가 §4c로 자기 서명 | `unverified` → **`machine-confirmed`**. 작성자 = 검증자 |
| **T3** | 문서화된 전체 흐름 `produce → enrich → (구조변경) produce → enrich` | ① `human-reviewed` [human:vavagirls] → ② produce 후 **`unverified`**, `verified`/`status`/`stale_after`/`sources` 전부 소실 → ③ §4c 적용 후 **`machine-confirmed`** [process:claude-enrich] |
| **T5** | `enriched_against == content_hash` 스킵 규칙에 frontmatter-only 변경 투입 | `resource`/`type` 변경 후에도 `content_hash` **불변**(A.1) → **여전히 스킵 대상**. 재보강 후보에 진입하지 못함 |

## C.2 Fixture별 판정

### F1 — Evidence insufficiency
- **INPUT / EVIDENCE CONDITION**: 스키마만 있고 profile·sample·관계가 없는 개념. 목적이 스키마로부터 결정되지 않음
- **SKILL INSTRUCTION USED**: §4 *"Ground every claim… Do not invent business meaning the evidence doesn't support. If the purpose is genuinely ambiguous, describe the structure and note what's uncertain rather than fabricating"* / §3 *"prefer to (re)produce with `--profile --sample` first"*
- **EXPECTED SAFE BEHAVIOR**: 보강을 보류하거나, 구조만 기술하고 불확실성을 표시. 증거보다 강한 단정 금지
- **OBSERVED / PROCEDURALLY IMPLIED**: 금지 규칙은 존재하나 **강제 수단이 없다(ⓒ)**. 보강 **거부 경로가 문서에 부재(ⓧ, C.0)** — 유일한 출구는 "설명 안에 불확실성을 적는다"이며 그 준수 여부를 확인할 필드도 도구도 없다. 반대로 구조적 **역인센티브**가 존재한다: §5 triage와 `--min` 게이트(§B)가 **채우는 쪽을 보상**하고, 채워진 뒤에는 §Idempotency 스킵 규칙이 `enriched_against == content_hash` 로 **동결**하며, §2 don't-clobber가 이후 수정을 **금지**한다
- **FAILURE MODE**: 근거 없는 단정이 1회 기입되면 **영구 동결**되고 이후 재검토 대상에서 제외됨
- **SEVERITY**: 높음 (knowledge correctness) · **DETECTABILITY**: **없음** — §B 실측상 coverage는 근거 없는 설명을 100% enriched로 계수
- **VERDICT**: **FAIL (unenforced + counter-incentivized)**

### F2 — Conflicting evidence
- **INPUT**: 서로 모순되는 두 `sources` 를 가진 개념
- **SKILL INSTRUCTION USED**: 해당 규칙 **없음** (C.0)
- **EXPECTED SAFE BEHAVIOR**: 충돌 보존, 임의 선택 금지, unresolved 상태 표현
- **OBSERVED / PROCEDURALLY IMPLIED**: 절차 부재(ⓧ). 지침은 *"Base every claim on evidence in the document"* 만 말하고 evidence가 서로 다툴 때를 규정하지 않는다. unresolved를 표현할 frontmatter 필드도 없다. 임의 선택이 기본 경로가 된다
- **FAILURE MODE**: 한쪽 evidence의 조용한 소거. 산출물은 단일하고 확신에 찬 설명으로 보인다
- **SEVERITY**: 높음 (evidence fidelity) · **DETECTABILITY**: **없음**
- **VERDICT**: **FAIL (rule absent)**

### F3 — Existing human verification
- **INPUT**: `verified: human:…` 와 사람이 쓴 description/body 를 가진 개념
- **SKILL INSTRUCTION USED**: §2 *"Do not overwrite a substantive, human- or source-authored description"* / §4c *"Preserve existing verifications: Append new verification events… without dropping existing entries"* / §5 *"Preserve `type`,`title`,`resource`,`timestamp` and the markdown body unchanged"*
- **EXPECTED SAFE BEHAVIOR**: 기존 설명·검증 보존. 사람 검증의 의미가 자동 수행으로 승격/변질되지 않음
- **OBSERVED / PROCEDURALLY IMPLIED**: 지침 자체는 **보존을 명시**하며 이 부분은 spec §5.2와 정합하다 — **지침의 결함이 아니다.** 그러나 실측(T3)상 `produce` 경로가 개입하면 사람 검증이 소실된 뒤 §4c가 기계 서명을 얹는다. 또한 don't-clobber는 `description` 만 보호하며 §4a 관계 산문·§4b 태그는 **기존 사람 검증 도장 아래에 새 내용이 추가**된다 — 사람이 보지 않은 내용을 사람의 서명이 담보하게 된다. 이 상황을 감지·하향할 규칙은 부재(ⓧ)
- **FAILURE MODE**: 사람 서명의 담보 범위가 조용히 확대됨. `produce` 개입 시 사람 서명 소실 후 기계 서명으로 대체
- **SEVERITY**: **치명** (trust state) · **DETECTABILITY**: **없음** (§A.3에서 lint/coverage 모두 clean 반환)
- **VERDICT**: **PARTIAL FAIL** — 보존 규칙은 존재하고 옳다. 담보 범위 확대에 대한 규칙이 없고, 소실 경로를 탐지하지 못한다

### F4 — Unsupported relationship
- **INPUT**: 이름상 관련 있어 보이나 스키마에 근거가 없는 두 개념
- **SKILL INSTRUCTION USED**: §4a *"Only describe edges the connector emitted — never fabricate a relationship the schema does not support"* / *"preserve the deterministic link list (the connector owns it)"* / *"never create the section when the connector did not"*
- **EXPECTED SAFE BEHAVIOR**: 근거 없으면 관계를 만들지 않음. 방향·카디널리티를 추정하지 않음
- **OBSERVED / PROCEDURALLY IMPLIED**: **본 Skill에서 가장 강하게 방어된 지점.** 링크 목록의 소유권이 connector에 있고 결정적으로 재생성되므로, 조작된 링크는 `produce` 재실행 시 diff로 드러난다 ⇒ **강제력 ⓑ(사람 검사 가능)**. 다만 카디널리티 산문(*"each order is placed by exactly one customer"*)은 자유 서술이며 §4a의 *"Ground cardinality in keys/uniqueness"* 는 ⓒ에 머문다
- **FAILURE MODE**: 링크 위조는 탐지 가능. 카디널리티 과단정은 탐지 불가
- **SEVERITY**: 중 · **DETECTABILITY**: 링크 **높음** / 산문 **없음**
- **VERDICT**: **PASS (링크)** / **WEAK (카디널리티 산문)**

### F5 — Misleading / noisy evidence
- **INPUT**: 우연한 상관이 강한 sample, 저카디널리티 컬럼
- **SKILL INSTRUCTION USED**: §3 *"a 2–3-distinct column is likely a flag, enum, or status"*, *"Treat these as primary, near-mechanical grounding"* / §4a *"If the direction is genuinely ambiguous, state the link factually and note the uncertainty — never invent a cardinality"*
- **EXPECTED SAFE BEHAVIOR**: observation과 inference의 분리, 과도한 규칙 확정 금지
- **OBSERVED / PROCEDURALLY IMPLIED**: 지침이 **추론을 적극 권장**한다(*"likely a flag/enum/status"*, *"min/max timestamps reveal the time span"*). 억제 규칙(*"note the uncertainty"*)은 존재하나 ⓒ이며, 산출물에서 관측과 추론을 구분해 기록할 **형식이 없다** — 둘 다 동일한 `description` 한 줄로 합쳐진다. rubric에도 해당 차원이 없다(C.0)
- **FAILURE MODE**: sample 상관이 business rule로 승격되어 단정문으로 고착
- **SEVERITY**: 높음 · **DETECTABILITY**: **없음**
- **VERDICT**: **FAIL (구조적으로 구분 불가)**

### F6 — Re-run / idempotency
- **INPUT**: 동일 evidence로 2회 이상 수행
- **SKILL INSTRUCTION USED**: §4b *"union, deduplicated, sorted… Re-running yields the same set"* / §4c *"Append new verification events"* / §Idempotency markers / 품질규칙 4 *"safe to re-run"*
- **EXPECTED SAFE BEHAVIOR**: 2회차가 실질적 no-op
- **OBSERVED / PROCEDURALLY IMPLIED**: description **PASS** (§2 don't-clobber + 스킵 규칙). tags **PASS** (합집합·정렬 명시). 관계 산문 **PASS** (*"add prose only for edges that lack a gloss"*). **`verified` FAIL** — T1 실측상 중복 제거 규칙이 지침에도 라이브러리에도 없어 동일 항목이 누적된다. spec §5.2는 *"Multiple entries capture **independent** checks"* 라 하므로, 중복 누적은 **독립 검증이 여러 번 있었다는 거짓 신호**가 된다
- **FAILURE MODE**: 검증 이력 부풀림 → 신뢰 과대표시
- **SEVERITY**: 중 · **DETECTABILITY**: **높음** (frontmatter 육안 확인 가능)
- **VERDICT**: **PARTIAL FAIL (`verified` 한정)**

### F7 — Changed evidence
- **INPUT**: 1차 보강 후 소스 스키마가 변경되어 기존 설명이 모순됨
- **SKILL INSTRUCTION USED**: §Idempotency *"A structural change bumps `content_hash`, so `enriched_against` no longer matches and the concept automatically re-enters the candidate set"* / §2 *"Do not overwrite a substantive… description unless the user explicitly asks"*
- **EXPECTED SAFE BEHAVIOR**: 모순된 기존 설명 식별, 이전 추론의 철회 또는 신뢰 하향
- **OBSERVED / PROCEDURALLY IMPLIED**: **두 규칙이 서로 막는다.** 구조 변경으로 후보에 재진입하지만, 재진입한 개념의 설명은 이미 substantive이므로 §2가 덮어쓰기를 금지한다. 실측(§2.2 CASE A)상 `MergeConcept`은 구조 변경 시 기존 description을 **무조건 보존**한다. 철회·하향 규칙은 부재(ⓧ, C.0). 추가로 T5 실측상 **frontmatter-only 변경은 `content_hash`를 바꾸지 않아 재진입조차 하지 않는다**
- **FAILURE MODE**: 스키마와 모순되는 설명이 설계상 영속. 신원이 바뀐 개념은 재검토 대상에서 영구 제외
- **SEVERITY**: **치명** (knowledge correctness) · **DETECTABILITY**: **없음**
- **VERDICT**: **FAIL**

### F8 — Trust boundary
- **INPUT**: machine 생성 evidence와 human 확인 evidence 혼재
- **SKILL INSTRUCTION USED**: §4c *"When performing automated machine enrichment, set `verified: {by: process:<agent-name>, at: …}`. This transitions the concept's derived trust tier from `unverified` to `machine-confirmed`"*
- **EXPECTED SAFE BEHAVIOR**: 작성 행위와 검증 행위의 분리 유지. 자동 수행만으로 신뢰 등급이 오르지 않음
- **OBSERVED / PROCEDURALLY IMPLIED**: T2 실측 — 설명을 **방금 작성한 바로 그 actor**가 서명해 `unverified → machine-confirmed`. 정본 SPEC §5.2는 *"`generated` records how the current content was **produced**… `verified` records who or what has **confirmed the content against its sources**. They are kept distinct **because who wrote a concept need not be who confirmed it**"* 라고 두 축의 분리를 명시한다. §4c는 이 분리를 붕괴시킨다. 독립 확인 없이 신뢰 등급이 상승하며, 이는 지침이 **지시하는** 동작이다
- **FAILURE MODE**: 자기서명으로 만들어진 신뢰 신호. T3와 결합하면 사람 검증 소실 직후 기계 서명이 얹혀, 문서가 **"검증됨"으로 보이는 상태**가 된다 — 부재(`unverified`, 탐지 가능)를 **거짓 양성(`machine-confirmed`, 탐지 불가)** 으로 전환한다
- **SEVERITY**: **치명** (trust state) · **DETECTABILITY**: **없음**
- **VERDICT**: **FAIL**

## C.3 요약

| Fixture | 강제력 | Severity | Detectability | Verdict |
|---|---|---|---|---|
| F1 근거 부족 | ⓒ + 역인센티브 | 높음 | 없음 | **FAIL** |
| F2 충돌 evidence | ⓧ 규칙 부재 | 높음 | 없음 | **FAIL** |
| F3 기존 사람 검증 | 규칙 존재(옳음) + ⓧ 범위확대 규칙 부재 | 치명 | 없음 | **PARTIAL FAIL** |
| F4 근거 없는 관계 | 링크 ⓑ / 산문 ⓒ | 중 | 링크 높음 | **PASS / WEAK** |
| F5 오도하는 evidence | ⓒ, 형식적 구분 불가 | 높음 | 없음 | **FAIL** |
| F6 재실행 멱등성 | tags ⓐ / verified ⓧ | 중 | 높음 | **PARTIAL FAIL** |
| F7 evidence 변경 | 규칙 상호 봉쇄 + ⓧ | 치명 | 없음 | **FAIL** |
| F8 신뢰 경계 | 지침이 위반을 **지시** | 치명 | 없음 | **FAIL** |

**"모델이 잘 판단하면 된다"에 의존하는 지점 (명시 요구 항목)**: F1의 불확실성 표기, F4의 카디널리티 근거화, F5의 관측/추론 분리. 이 셋은 준수 여부를 기록할 형식도, 검사할 도구도 없다.

**책임 범위 밖 (N/A)**: `verified`·`sources`·`status`의 **소실** 자체는 `okf-go`(§2.2, §A.2)의 문제이며 본 Skill에 계상하지 않는다. 본 Skill에 계상하는 것은 그 위에 **기계 서명을 얹어 거짓 양성을 만드는 §4c**다.

## C.4 재판정

| Asset | 기존 | **재판정** |
|---|---|---|
| **`okf-enrich`** | EXTEND (부록 B) | **EXTEND (유지, 근거 교체 및 범위 축소)** |

**유지 근거**: 문서의 다수 절차는 실측·대조로 건재함이 확인된다 — §4a 관계 규율(F4 PASS, 링크 소유권이 connector에 있어 위조가 **탐지 가능**), §4b 태그 합집합 멱등성(PASS), §2 don't-clobber와 §5 surgical write(F6 description PASS), 근거 수집 절차(§3), glossary 재사용·triage·batching. 이들은 대체 불가한 procedural value이며 폐기 대상이 아니다.

**하향하지 않은 이유(REFERENCE가 아닌 이유)**: 결함이 문서 전반에 스며든 것이 아니라 **분리 가능한 소수 지점에 집중**된다 — §4c(신뢰 서명), 그리고 부재한 세 규칙(충돌 처리·추론 철회·보강 보류). 나머지 절차는 이들과 독립적으로 성립한다. 즉 **안전한 확장 지점이 식별된다**(§A.4의 point-9 기준 충족). 이는 `okf-go`(§A.4.1)와 성격이 다르다 — 거기서는 훼손 경로가 upstream이 의도한 설계 계약 자체였다.

**단, 판정의 무게는 부록 B 때보다 무거워졌다.** B에서의 하향 사유는 "Verification이 완성도만 센다"였다. 이제는 그에 더해 **§4c가 부재하는 신뢰를 적극적으로 날조**하며(F8), **F1·F5·F7의 실패가 전부 탐지 불가**이고, **§2+§Idempotency+`--min` 게이트가 결합해 1회의 잘못된 보강을 영구 동결**시킨다(F1). 무수정 채택은 명확히 불가하다.

## C.5 본 부록의 한계

- 자가수행 채점을 배제했으므로, **실제 에이전트가 이 지침으로 산출한 description의 품질 분포는 측정하지 않았다.** 본 부록이 판정한 것은 *지침의 강제력과 실패의 탐지 가능성*이다.
- `eval/fixtures/cases.yaml` 의 라벨 baseline을 실행해 회귀 점수를 재현하지 않았다(LLM-judge 비결정성 + 자가채점 배제 원칙).
- `okf-git`/SQL connector 산출물이 필요한 `Comment` 컬럼 보강 경로는 미검증.
- 다른 asset 판정은 본 부록에서 변경하지 않았다.


---
---

# 부록 D. `okf-go` fork 경제성 실험

> **모드 전환 선언.** §1–부록C는 "수정하지 않는다 / 수정 방법을 제안하지 않는다" 원칙 아래 수행된 **evidence·판정** 작업이다. 본 부록은 사용자 지시에 따라 그 제약이 **이 범위에 한해 해제**된 **타당성 측정** 작업이다. 목적은 결함을 고치는 것이 아니라 **"작은 fork로 이 생태계를 가져다 쓸 수 있는가"의 비용을 숫자로 확정**하는 것이다.
>
> 원본 clone(`repos/okf-skills` @ `9740e89`)은 evidence anchor이므로 **손대지 않았다**(실험 종료 시점 `dirty=0` 재확인). 모든 작업은 별도 사본 `fork/okf-skills` 에서 수행했다.

## D.0 측정 설계 — 왜 "diff 크기"만으로는 답이 안 되는가

fork 경제성은 diff 줄 수가 아니라 아래 5개로 결정된다. 그래서 결함별로 이 5개를 각각 측정했다.

1. LOC / 파일 수
2. **공개 API 파손 여부** (12개 skill + okf-mcp 가 리터럴로 `Frontmatter` 를 생성한다)
3. upstream 테스트 유지 여부
4. **기존 데이터 무효화 여부** (코드 diff에 잡히지 않는 비용)
5. 의존 모듈 재빌드·재테스트 결과

## D.1 계측기 — boundary-fixture 회귀 하네스

산문 판단을 배제하기 위해, 앞선 부록들의 fixture를 **기계적 단언 18개**로 고정한 하네스를 별도 모듈로 작성하고 원본/fork에 동일하게 적용했다 (`harness/main.go`, `replace` 대상만 교체).

**BASELINE (원본 @`9740e89`): 10 PASS / 8 FAIL**

| ID | 항목 | baseline |
|---|---|---|
| B1 | encoding fidelity (한글/emoji/quote) | PASS |
| B2 | spec 정의 필드 왕복 | PASS |
| B3 | unknown/custom key 보존 | **FAIL** |
| B4 | enrichment 보존 (description) | PASS |
| B5 | provenance 보존 (sources 상세) | **FAIL** |
| B6 | trust 보존 (verified) | **FAIL** |
| B6b | lifecycle 보존 (status/stale_after) | **FAIL** |
| B7 | 증분: 무변경 ⇒ `changed==false` | PASS |
| B8 | 관계 렌더링 결정성 | PASS |
| B9 | 구조 해시 결정성 | PASS |
| B9b | 신원 변경 탐지 (resource/type/title) | **FAIL** |
| B10 | `stale_after` 정본 instant (과거) | **FAIL** |
| B10b | `stale_after` 날짜형 (과거) | PASS |
| B10c | 정본 instant (미래) ⇒ 안 낡음 | PASS* |
| B11 | frontmatter 부재 ⇒ error | PASS |
| B11b | YAML 파손 ⇒ error | PASS |
| B12 | 이형 `verified` 노출 (무성 소멸 금지) | **FAIL** |
| B13 | 쓰레기 `stale_after` 가 '신선'으로 흡수 금지 | **FAIL** |

> **계측기 자체에 대한 정직한 지적**: baseline의 B10c는 **가짜 PASS**다. 미래 instant가 "안 낡음"으로 나온 것은 파싱 실패(`false`)가 우연히 정답과 겹친 결과다. fork 적용 후에는 RFC3339 파싱이 성공해 **진짜 PASS**가 된다.

## D.2 단계별 적용과 회복 곡선

| Tier | 내용 | 변경 | upstream tests | 하네스 |
|---|---|---|---|---|
| — | baseline | — | PASS | **10 / 18** |
| **T1** | `IsStale`: RFC3339 instant 수용, 파싱 실패 ⇒ **fail-closed(stale)** · `VerifiedList.UnmarshalYAML`: 예상 밖 노드 ⇒ **error 반환** | `okf.go` +11/−6 | **PASS** | **13 / 18** |
| **T2** | `Frontmatter.Extra map[string]interface{}` (`yaml:",inline"`, `json:"-"`) — 미모델링 키 보존 | `okf.go` +5 | **PASS** | **14 / 18** |
| **T3** | `MergeConcept`: fresh 가 비어 있을 때 `Verified`/`Sources`/`UsageWindow`/`Status`/`StaleAfter` 이월. `Extra` 는 당초 맵 전체 조건부 이월이었고 **부록 H(P4b)에서 키 단위 병합으로 정정** | `hash.go` +20 | **PASS** | **17 / 18** |
| **T4** | `ConceptStructuralHash`: `type`/`title`/`resource` 를 해시 입력에 포함 | `hash.go` +12/−2 | **PASS** | **18 / 18** |

**최종: 2개 파일, +48 / −8 (patch 100줄).** 변경 지점은 5개 함수/타입뿐 — `ConceptStructuralHash`, `MergeConcept`, `VerifiedList.UnmarshalYAML`, `Frontmatter`, `IsStale`.

## D.3 blast radius 실측 — 공개 API 파손 여부

`go.work` 의 16개 모듈 전체를 fork okf-go 위에서 빌드·테스트했다.

```
BUILD: OK=16  FAIL=0
TEST : OK=16  FAIL=0
```
대상: `okf-go`, `okf-mcp`, `skills/okf-{bigquery,csv,fs,git,graphql,lint,mongodb,mysql,openapi,postgresql,sqlite,viz}`, `tests`, `tools/scaffold`

**⇒ 소스 수정 0건, `go.mod` 수정 0건, 계약 연쇄변경 0건.** 정지 규칙("계약 연쇄변경이 튀어나오면 접는다")에 걸리지 않았다.

> **측정 오류 정정**: 1차 측정에서 4개 모듈이 FAIL로 보였으나, 이는 의존성 다운로드 로그를 출력으로 판정한 **내 계측 오류**였다. 종료코드 기준으로 재측정해 정정했다. 위 숫자는 재측정값이다.

## D.4 end-to-end 검증 (단위 하네스가 아닌 실제 connector 바이너리)

fork 로 `okf-fs` 를 빌드해 §3과 **동일한 시나리오**를 재실행했다 — 번들 생성 → 사람 검증·provenance·lifecycle·확장 키 주입 → 원본 구조 변경 → 재 produce.

| 주입 항목 | §3 (원본) | **fork** |
|---|---|---|
| `description` (사람 작성) | 보존 | 보존 |
| `verified` (human:vavagirls) | **소실** | **보존** |
| `sources` (id/author/usage_count) | **소실** | **보존** |
| `status: stable` | **소실** | **보존** |
| `stale_after` | **소실** | **보존** |
| `gcore_evidence_id` (확장 키) | **소실** | **보존** |

## D.5 T4 의 마이그레이션 비용 — 실측

해시 정의 변경은 **기존 번들의 모든 `content_hash` 를 무효화**한다. 코드 diff에 잡히지 않는 비용이므로 직접 측정했다.

절차: 원본 okf-fs 로 5개 개념 번들 생성 → 각 개념에 human 검증 주입 → **소스 변경 없이** fork okf-fs 재실행.

| 실행 | 결과 |
|---|---|
| 업그레이드 직후 1회차 | **5/5 개념 재작성** (전 개념이 "구조 변경"으로 판정) |
| 2회차 | **5/5 "Unchanged, preserved"** |
| 3회차 | **5/5 "Unchanged, preserved"** |
| 1회 churn 이후 `human:vavagirls` 잔존 | **5/5 보존** |

**⇒ 비용은 "1회 전면 재작성 후 즉시 안정화"로 한정되며, 그 churn 이 신뢰·provenance 를 파괴하지 않는다.** 단 이는 **T3가 먼저 적용되어 있기 때문**이다 — T4를 T3 없이 단독 적용하면 동일 churn 이 §2.2의 전면 소실을 1회 강제한다. **적용 순서가 안전성의 전제다.**

부수 비용: `enriched_against` 도 전 개념에서 불일치가 되어 **1회 전면 재보강 후보화**된다(토큰 비용). 데이터 훼손은 아니다.

## D.6 fork로 해결되지 **않는** 것 (명시)

- **skill 계층 결함은 그대로다.** 부록 C의 F1(근거 부족 시 보류 경로 부재)·F2(충돌 처리 부재)·F5(관측/추론 미분리)·F7(철회 규칙 부재)·**F8(§4c 자기서명)** 은 전부 지침 문서의 문제이며 라이브러리 fork로 해결되지 않는다.
- 다만 C의 **T3 복합 시나리오는 절반이 해소**된다: 파괴 단계(`produce` 가 사람 검증을 지우는 것)는 fork 가 막는다. 남는 것은 **§4c 가 작성자 자신의 서명으로 신뢰 등급을 올리는 부분**이다.
- **workspace 밖 설치 경로**: `go.work` 가 로컬 매핑을 하므로 workspace 내부 빌드는 무비용이지만, SKILL.md 가 안내하는 `go install github.com/xSAVIKx/…@v0.9.0` 독립 설치 경로는 모듈 경로 변경 또는 `replace` 가 필요하다. 이는 우리가 소유해야 하는 릴리즈 경로다.
- **상류 spec 드리프트 리스크는 지속된다.** §7에서 확인했듯 GCP 는 v0.2 를 **버전 번호 없이 제자리 변경**한 전례가 있다. fork 는 이 리스크를 없애지 않는다.
- `tests/` 통합 테스트는 DB 서비스가 없을 때 skip 되는 구조다. **라이브 DB 대상 검증은 수행하지 않았다.**
- T4 의 해시 입력으로 `type`/`title`/`resource` 를 선택한 것은 **본 실험의 선택**이며, 6개 SQL/VCS connector 가 이 세 필드를 신원으로 쓰는지에 대한 전수 검증은 하지 않았다.

## D.7 결론 — 경제성 판정

| 질문 | 실측 답 |
|---|---|
| 작은 fork로 13개 경계 fixture를 회복할 수 있는가 | **가능. 2파일 +48/−8 로 18/18** |
| upstream 테스트가 유지되는가 | **유지 (okf-go 및 16개 모듈 전부)** |
| 공개 API 계약 연쇄변경이 발생하는가 | **발생하지 않음 (16/16 무수정 빌드·테스트)** |
| 숨은 데이터 비용이 있는가 | **있음** — T4의 1회 전면 재작성. 단 T3 선행 시 **비파괴·즉시 안정화** |
| 정지 규칙(계약 연쇄변경) 발동 | **미발동** |

**⇒ `okf-go` 를 DROP-AS-IS 로 판정한 §A.4 는 유지된다** — 무수정 채택은 여전히 불가하다. 그러나 본 실험은 **"수정 비용이 작고 국소적이며 연쇄되지 않는다"** 는 별개의 사실을 확정한다. 이 둘은 모순이 아니다: DROP-AS-IS 는 *as-is 채택 가능성*에 대한 판정이고, D는 *fork 채택 경제성*에 대한 측정이다.

> 본 부록은 경제성 측정까지만 수행한다. 어느 자산을 실제로 fork 하여 운영에 넣을지, skill 계층 결함(D.6)을 어떻게 다룰지에 대한 결정과 설계는 포함하지 않는다.


---
---

# 부록 E. 채택 결정 및 fork 실행 기록

> 부록 D는 경제성 **측정**까지였다. 본 부록은 그 측정에 근거해 내려진 **채택 결정과 실행 결과**를 기록한다.

## E.1 결정

| 항목 | 결정 |
|---|---|
| 방향 | **fork 하여 고쳐 쓴다** |
| 위치 | `C:\workspace\prj20060203\okf_series\okf-skills-fork\` — 독립 git repo (`upstream` 원격 등록 + 이력 fetch 완료 → `git diff upstream/master` 즉시 가능) |
| 모듈 경로 | `github.com/xSAVIKx/okf-skills/...` **당분간 유지** (workspace 내부 빌드 무비용). 독립 배포가 필요해질 때 13개 `go.mod` 기계적 변경 |
| base | upstream `9740e89309b0a10b0e847ac533cc33a7b4513f5c` |

## E.2 실행 결과

| 항목 | 결과 |
|---|---|
| 패치 | `okf-go` 2파일 **+48/−8** (P1~P5, `FORK.md` 참조) |
| 회귀 게이트 | `okf-go/boundary_test.go` — 13개 경계 요구를 **14개 테스트 함수**로 고정. scratch 하네스에서 정식 테스트로 승격 |
| **게이트 반증력 검증** | 동일 스위트를 **패치 없는 upstream** 에 적용 → **7개 함수 FAIL** (단언 10건). fork → **14/14 PASS**. 통과만 하는 스위트가 아님을 확인 |
| 모듈 재검증 (새 위치) | **16/16 TEST OK** |
| 저장소 정리 | 커밋된 connector ELF 바이너리 6개 + 내 빌드 산출물 14개 제거, `.gitignore` 추가 → 작업 트리 **182MB → 2.5MB** (저장소 전체는 상류 이력 팩 포함 약 60MB) |
| 커밋 | `6e3d8bd` (247 files, initial) |
| 원본 clone 무결성 | `dirty=0 @9740e89` — evidence anchor 훼손 없음 |

## E.3 판정과의 관계

§A.4의 **`okf-go` = DROP-AS-IS 는 철회하지 않는다.** 그 판정은 *무수정 채택 가능성*에 대한 것이고 여전히 옳다 — 실제로 수정 없이는 쓰지 않았다. 부록 D·E가 확정한 것은 별개 사실이다: **수정 비용이 작고(2파일 +48/−8), 국소적이며(함수·타입 5개), 계약 연쇄변경을 일으키지 않는다(16/16 무수정 빌드).**

## E.4 남은 작업 (fork로 해결되지 않음)

- **`okf-enrich` skill 계층**: §4c 자기서명(F8), 보강 보류 경로 부재(F1), 충돌 처리 부재(F2), 추론 철회 규칙 부재(F7), `verified` 중복 제거 규칙 부재(F6). 판정 **EXTEND** 유지. *P4 덕분에 C.1-T3 복합 시나리오의 파괴 단계는 차단되었으나, 자기서명 자체는 남아 있다.*
- **`okf-viz coverage`**: 완성도 계수기이지 정확성 검사가 아님. 신뢰 무결성 게이트로 사용 불가. 판정 **KEEP(기준 한정)** 유지.
- **`okf-producer-generator`**: 드리프트된 `okf-SPEC.md` 동봉, 등록 체크리스트에 placeholder 패턴 등록 단계 누락. 판정 **REFERENCE** 유지.
- **미검증 connector 10종**: 여전히 `NOT TESTED`. fork 위에서 빌드·테스트는 통과하나(16/16), 각자의 고유 결함은 검증하지 않았다.
- **상류 spec 드리프트**: GCP가 v0.2를 버전 번호 없이 제자리 변경한 전례(`3dc3029`)가 있다. fork가 이 리스크를 제거하지 않는다. spec 정본 repo도 함께 추적해야 한다.
- **독립 설치 경로**: `go install …@v0.9.0` 는 여전히 upstream을 가리킨다. 모듈 경로 변경 시점은 미정.


---
---

# 부록 F. F8 (`okf-enrich §4c` 자기서명) — policy path 교정

> **범위 고정.** 부록 C의 F8 하나만 다룬다. F1/F2/F5/F7 및 기타 enrichment 품질 문제는 상태 변경 없이 유지한다.
> **분류.** F8을 SPEC 위반으로 전제하지 않는다. 현재 분류는 *"SPEC이 허용할 수 있으나 enterprise trust 목적에는 부족한 Skill policy"* 다.
> **검증 방법.** instruction-only Skill이므로 LLM 행동이나 자기평가를 테스트하지 않는다. `instruction → 지시된 frontmatter → okf-go GetTrustTier → trust tier` 결정적 체인만 사용한다. 에이전트를 실행하지 않았고 모델 출력을 채점하지 않았다.

## F.0 고정된 상태

| 항목 | 상태 |
|---|---|
| `okf-go` fork | **CLOSED** — 공식 regression gate는 `okf-go/boundary_test.go` **14/14**. scratch 단계의 18/18은 이후 기준으로 사용하지 않는다 |
| `okf-oracle` integration | **HOLD-ENV** — 실물이 이 PC에 없다. 회사 PC에서 read-only 로 `go.mod` / `go.work` / `replace` / import path / 빌드가 실제로 선택하는 `okf-go` 를 확인하기 전까지 "fork 적용 완료" 라고 주장하지 않으며, 추측하지도 않는다. *(사용자 진술: "go.mod 없어, 거기까지 안 갔다" — **미검증 진술로만 기록**한다. 본 문서는 직접 확인한 것만 근거로 삼으므로 이 진술로 상태를 변경하지 않는다. 확인 후 갱신.)* (참고: 이 PC 의 `C:\workspace\prj20060203\okf-loom` 은 순수 Python — py 3 / go 0 / `go.mod` 0 / `okf-go` 참조 0건. **이는 okf-loom 에 대한 관측이며 okf-oracle 에 대한 근거가 아니다.**) |

> 이 HOLD 로 인해 최종 producer 가 `okf-go` fork 를 물지 않을 가능성이 열려 있다. 그 경우 fork 에서 넘어가는 것은 라이브러리가 아니라 그것이 만족한 **행동**이며, 해당 파생 요구사항 초안은 리포트 밖의 별도 문서 `DERIVED-REQUIREMENTS.md` 에 있다 (검증 리포트의 일부가 아니며 확정된 요구사항도 아니다).

`§4c` 검증은 이 HOLD와 독립이다. §4c가 기대는 전제 *"produce가 기존 verified/trust/provenance를 파괴하지 않는다"* 는 fork **P4**에서 이미 검증되었다.

## F.1 Baseline

**현재 instruction (patch 전, `skills/okf-enrich/SKILL.md`)**

- `L114` §4c 첫 불릿 — *"**Machine enrichment sign-off**: When performing automated machine enrichment, set frontmatter `verified: { by: "process:<agent-name>", at: "<ISO8601>" }` … This transitions the concept's derived trust tier from `unverified` to `machine-confirmed`."*
- `L126` §5 write-back — *"Set the frontmatter `description` field and **append `verified: { by: "process:<agent>", at: "<timestamp>" }`**…"*

⇒ 자기서명 경로가 **두 곳**에서 지시된다.

**Fixture / 결과** (actor A = `process:claude-enrich`)

| ID | 상태 | `GetTrustTier` |
|---|---|---|
| **B0** | A가 enrichment 수행 → A가 자신을 `verified` 에 기록, 독립 verification 없음 | **`machine-confirmed`** |

⇒ **F8 baseline 결정적으로 재현됨.** 동일 actor의 자기서명만으로 trust 상승이 발생한다.

## F.2 Patch

**SPEC 근거** (정본 `GoogleCloudPlatform/open-knowledge-format` @ `ad30107`)

- §5.2 — *"`generated` records **how the current content was produced**. `verified` records **who or what has confirmed the content against its sources or `resource`**. They are kept distinct **because who *wrote* a concept need not be who *confirmed* it**."*
- §5.2 — `generated.by`: **REQUIRED** within `generated`
- §5.3 — trust tier는 **`verified` 에서만** 도출된다

⇒ enrichment 수행은 §5.2 정의상 *produced* 에 해당하므로 `generated.by` 가 그 기록 위치다. 동일 actor를 `verified` 에 넣는 것은 §5.2가 분리해 둔 두 역할을 결합시킨다.

**변경 (2줄)**

| 위치 | 변경 |
|---|---|
| `L114` | `verified` 자기 기록 지시 → **`generated: { by, at }` 기록** 지시로 교체 + *"Do not add the enriching actor to `verified`"* 명시 + 근거(§5.2) 인용 + *"enrichment 만으로는 tier가 `unverified` 에 머문다"* 명시 |
| `L126` | `append verified: {…}` 절 삭제 → `generated` 설정으로 교체 + *"do not write a `verified` entry for your own enrichment"* |

**변경하지 않은 것**: `L115` Human sign-off (사람이 독립적으로 확인하는 정당한 경로), `L116` Preserve existing verifications.

## F.3 Patched Result

패치된 §4c가 요구하는 상태를 fixture로 직접 구성:

| ID | 상태 | `GetTrustTier` |
|---|---|---|
| **P0** | A가 enrichment → `generated.by = A`, `verified` 항목 없음 | **`unverified`** |

⇒ **PASS.** 패치된 §4c의 정상 지시 경로를 따를 때, actor A 자신의 enrichment만으로는 독립 verification trust가 생성되지 않는다.

## F.4 Regression

| ID | 상태 | `GetTrustTier` | 기대 |
|---|---|---|---|
| **R1** | A enrich → **human B** verify | `human-reviewed` | 유지 |
| **R2** | A enrich → **독립 process B** verify | `machine-confirmed` | 유지 |
| **R3** | 기존 verification 보존 + 독립 B 추가 | `human-reviewed` | 유지 |
| **C1** | `verified` 없음 (대조군) | `unverified` | 유지 |

⇒ 정당한 independent verification 경로의 trust behavior가 **전부 보존**되었다.

## F.5 Cost

| 항목 | 값 |
|---|---|
| 변경 파일 | **1** (`skills/okf-enrich/SKILL.md`) |
| 변경 라인 | **+2 / −2** |
| 다른 component 변경 | **없음** — `okf-go` 변경 파일 0개. `GetTrustTier`·producer·lint·validator·SPEC 모두 무변경 |
| 회귀 게이트 | `okf-go/boundary_test.go` **14/14 PASS** (변경 없음 확인) |
| 커밋 | `b6734ca` |

**부수 확인 (읽기 전용, 변경 없음)**: 저장소 내 다른 자기서명 지시 경로는 없다. `okf-reader/SKILL.md:75` 는 **판독** 지시이고, `okf-producer-generator/okf-SPEC.md:985` 는 벤더링된 spec 예시(`process:finance-nightly` = 독립 프로세스)다.

## F.6 Remaining Blockers

- **`Self-sign mechanical detection gate absent`** — §4c는 instruction이므로, 에이전트가 지시를 **위반하고** 직접 `verified` 를 쓰는 것을 기계적으로 막지 못한다. `generated.by == verified.by` 같은 self-sign 상태를 결정적으로 탐지하는 게이트는 존재하지 않는다. **follow-up candidate: lint/validator layer.** (이번 범위에서 okf-go·lint·validator 수정은 금지되어 손대지 않았다.)
- 부록 C의 나머지 항목은 **상태 변경 없이 유지**: F1 근거 부족 시 보류 경로 부재 · F2 충돌 evidence 처리 부재 · F5 관측/추론 미분리 · F6 `verified` 중복 제거 규칙 부재 · F7 추론 철회 규칙 부재.
- `okf-viz coverage` = 완성도 계수기 (부록 B), 판정 **KEEP(기준 한정)** 유지.
- `okf-enrich` 자산 판정은 **EXTEND 유지** — F8 policy path는 교정되었으나 F1/F2/F5/F7이 미해결이므로 무수정 채택은 여전히 불가하다.
- `okf-oracle` integration **HOLD-ENV** 유지 (F.0). 회사 PC read-only 확인 전까지 상태 변경 없음.

## F.7 최종 판정 (2층 분리)

| 층 | 판정 |
|---|---|
| **F8 Skill policy path** | **REMEDIATED** |
| **F8 mechanical enforcement** | **OPEN / NOT IN SCOPE** |

> 정확한 성공 표현: **The default §4c self-signing instruction path is removed and deterministically verified, while mechanical detection/enforcement against instruction violation remains open at the lint/validator layer.**
>
> `F8 CLOSED` 로 표기하지 않는다. **Policy path closed ≠ Enforcement closed.**


---
---

# 부록 G. F8 mechanical enforcement — self-sign policy gate

> **범위.** F8의 기계적 강제만 다룬다. F1/F2/F5/F7 등 다른 항목으로 확장하지 않았다.
> **분류 원칙.** `generated.by == verified.by` 는 **SPEC 위반으로 증명된 바 없다.** OKF v0.2 §5.3 은 trust tier 를 `verified` 에서 도출할 뿐이고, 한 actor 가 `generated` 와 `verified` 양쪽에 나타나는 것을 금지하지 않는다. 이 게이트에 걸리는 번들도 **여전히 conformant** 하다. 따라서 이번에 닫는 것은 **self-sign mechanical policy gate** 이지 **OKF conformance rule** 이 아니다.

## G.1 구조 확인 (읽기 전용, 선행)

| 계층 | 소유 | 내용 |
|---|---|---|
| SPEC conformance | `okf-go/lint.go` `ScanBundle` → `LintReport.Conformance` | 규칙 5종: `root-index-extra-keys`, `root-index-okf-version`, `subdir-index-frontmatter`, `concept-unparseable`, `concept-missing-type` |
| 게이팅 | `skills/okf-lint/main.go` `gateFailures(rep, opts)` | 순수 함수. 대부분 opt-in 플래그 |

⇒ **분리 가능**. `okf.ReadConceptDoc` 이 exported 이므로 policy 층을 `okf-lint` 안에만 추가할 수 있고 `okf-go` 는 손대지 않아도 된다.

## G.2 구현 — 분리 보장 장치

| 장치 | 내용 |
|---|---|
| 타입 분리 | `PolicyFinding` 별도 타입. `LintReport.Conformance` 에 **구조적으로 들어갈 수 없다** |
| 규칙 id 네임스페이스 | `policy-self-signed-verification` — `okf.Rule*` 와 혼동 불가 |
| opt-in | `-policy-no-self-sign` (기본 `false`) |
| 기본 동작 불변 | 출력은 stderr. **stdout 은 플래그 유무와 무관하게 바이트 동일** (실측 확인) |
| 출력 라벨 | `trust policy violations (not spec conformance)` |
| `okf-go` | **변경 파일 0개** |

파일: `skills/okf-lint/policy.go` (신규), `policy_test.go` (신규), `main.go` +17, `schema.go` +1.

## G.3 Deterministic fixtures (8) — 에이전트 미실행

| 상태 | 결과 |
|---|---|
| `generated.by=A`, `verified=A` | **FAIL** |
| `generated.by=A`, `verified=[A, human]` | **FAIL** — 독립 검증자가 있어도 자기서명 기록은 남아 있으므로 가려지지 않는다 |
| `generated.by=A`, `verified=B` (독립 process) | PASS |
| `generated.by=A`, `verified=human` | PASS |
| `generated.by=A`, `verified` 없음 | PASS |
| `generated.by` 없음, `verified=A` | PASS — 비교 대상이 없으므로 추측하지 않는다 |
| 예약 파일 (`index.md`, `log.md`) | 검사 대상 아님 |
| 훼손된 concept | 무시 — conformance 소관이며 policy 위반으로 전환하지 않는다 |

`go test -run TestPolicy` → **8/8 PASS**

## G.4 End-to-end (실제 바이너리)

| 검사 | 결과 |
|---|---|
| 플래그 **없이** self-sign 번들 | exit **0** — 기본 semantics 불변 |
| 플래그 **있이** self-sign 번들 | exit **1**, `[policy-self-signed-verification] c.md: …` |
| 플래그 있이 independent 번들 | exit **0** |
| stdout 바이트 비교 (플래그 유무) | **동일** |
| conformance 회귀 — 훼손 번들 (플래그 없음) | exit **1** |
| conformance 회귀 — 정상 번들 (플래그 없음) | exit **0** |

## G.5 회귀

- **Producer boundary gate (okf-go) = 14/14 PASS** — 무변경
- `okf-lint` 자체 스위트 PASS
- 워크스페이스 **16/16 모듈 PASS**
- `okf-go` 변경 파일 **0**
- 커밋 `b79bee7`

## G.6 게이트 계수 — 계층 분리

두 게이트를 합산하지 않는다. 계층이 다르다.

| 게이트 | 대상 | 개수 |
|---|---|---|
| **Producer boundary gate** | `okf-go` 행동 계약 | **14** |
| **Trust policy enforcement gate** | `okf-lint` 정책 강제 | **+1** |

## G.7 F8 최종 상태

| 층 | 판정 |
|---|---|
| **F8 Skill policy path** | **REMEDIATED** (부록 F) |
| **F8 mechanical enforcement** | **REMEDIATED — opt-in policy gate 로 한정** |

정확한 표현: **지시 위반으로 작성된 self-sign 상태는 이제 `okf-lint -policy-no-self-sign` 으로 결정적으로 탐지되며, 이는 SPEC conformance 가 아니라 명시적으로 분리된 trust policy 층에서 이루어진다. 기본 lint semantics 는 변경되지 않았다.**

**잔여 한계 (명시)**
- 게이트는 **opt-in** 이다. 켜지 않으면 탐지되지 않는다 — 운영에서 실제로 켜는 것은 별도 문제다.
- 번들을 **검사하는 시점**에만 작동한다. 작성 시점에 막는 것이 아니다.
- `generated.by` 가 없는 concept 은 비교 대상이 없어 판정하지 않는다.

## G.8 이번에 하지 않은 것

부록 C 의 나머지는 **상태 변경 없음**: F1 보강 보류 경로 부재 · F2 충돌 evidence 처리 부재 · F5 관측/추론 미분리 · F6 `verified` 중복 제거 규칙 부재 · F7 추론 철회 규칙 부재.
`okf-enrich` 판정 **EXTEND 유지**. `okf-oracle` **HOLD-ENV 유지**. connector 10종 **NOT TESTED 유지**.

**upstream PR 은 이번 작업과 무관하다.** 본 게이트는 우리 정책이므로 PR 대상이 아니다. PR 가치가 있는 것은 P1(정본 `stale_after` instant 미수용 + fail-open)과 P2(이형 `verified` 무성 소멸) 두 건이며, 별도 판단으로 남긴다.


---
---

# 부록 H. C5 — 확장 키 무성 소실 (P4b)

> **범위.** 부록 G 이후 §5(F1/F5 요구사항 정의) 중 실측된 `MergeConcept` 결함 하나만 닫는다. 새로운 정책 설계가 아니다. F2/F7 등으로 확장하지 않았다.

## H.1 Baseline — C5 FAIL

**관측**: existing concept 에 policy/extension 마커가 있고 fresh concept 에 **다른 확장 키가 하나라도** 있으면, 기존 `Extra` 전체가 대체되어 마커가 **무성 소실**한다.

**root cause**: P4 의 이월 로직이 **all-or-nothing** 이었다.

```go
if len(merged.Frontmatter.Extra) == 0 {
    merged.Frontmatter.Extra = existing.Frontmatter.Extra
}
```

⇒ 확장 키 보존이 **"어떤 connector 도 확장 키를 방출하지 않는다"는 조건**에 걸려 있었다. 그 조건이 깨지면 오류도 diff 도 없이 사라진다 — 이 fork 가 닫으려던 것과 정확히 같은 모양의 무성 소실이다.

**baseline 측정 결과** (fork, P4b 적용 전)

| Fixture | 결과 |
|---|---|
| C5-1 existing-only 키가 fresh 키와 함께 생존 | **FAIL** |
| C5-2 동일 키 충돌 → fresh 우선 | PASS (wholesale 치환이 우연히 같은 결과를 냈다) |
| C5-3 policy 키가 아닌 existing-only 키도 이월 | **FAIL** |

> **baseline 기준 주의.** C5 는 **unpatched upstream 에서 실행 자체가 불가능**하다 — 거기에는 `Frontmatter.Extra` 가 없어 테스트가 컴파일되지 않는다. 따라서 C5 의 의미 있는 baseline 은 upstream 이 아니라 **이 fork 의 P4b 이전 상태**다. 이 경계는 **P3 때문에 비로소 표현 가능해졌다.**

## H.2 Minimal merge rule (P4b)

확장 키를 **키 단위로** 병합한다.

1. fresh 에 있는 키는 **fresh 값이 우선**한다
2. fresh 에 없고 existing 에만 있는 키는 **이월**한다
3. 맵 전체를 **wholesale replacement 하지 않는다**

즉 `merged.Extra = existing.Extra + fresh.Extra (fresh wins on collision)`.

**구현 주의**: `merged := fresh` 구조상 `merged.Extra` 는 `fresh` 의 맵과 같은 참조다. 새 맵을 할당해 **어느 입력의 맵도 aliasing·변형하지 않는다.**

**의도적으로 풀지 않은 것**: producer 와 agent/policy 가 같은 키를 동시에 소유하는 문제는 이 병합 알고리즘에서 해결하지 않는다. `fresh wins` 는 **충돌 fallback 규칙이지 ownership model 이 아니다.** 네임스페이스 제약은 `DERIVED-REQUIREMENTS.md` 에 기록했다.

## H.3 Patched result

| Fixture | 결과 |
|---|---|
| C5-1 existing-only 키가 fresh 키와 함께 생존 | **PASS** |
| C5-2 동일 키 충돌 → fresh 우선 | **PASS** |
| C5-3 policy 키가 아닌 existing-only 키도 이월 | **PASS** |

scratch fixture 로 두지 않고 `okf-go/boundary_test.go` 의 정식 회귀 케이스로 승격했다 — `TestBoundaryExtraKeyLevelMerge` (서브테스트 3개).

## H.4 Regression

| 항목 | 결과 |
|---|---|
| **Producer boundary gate** | **14 → 15** 함수, **15/15 PASS** |
| upstream `okf-go` tests | PASS |
| 워크스페이스 16개 모듈 | **16/16 PASS** |
| F8 trust policy gate (`okf-lint`) | PASS |

**게이트 계수**: Producer boundary gate **15** / Trust policy enforcement gate **+1**. 계층이 다르므로 합산하지 않는다.

## H.5 Cost

| 항목 | 값 |
|---|---|
| 수정 파일 | **2** (`okf-go/hash.go`, `okf-go/boundary_test.go`) |
| 소스 변경 | `hash.go` **+14 / −2** (테스트 포함 시 총 +76 / −2) |
| `MergeConcept` 외 소스 변경 | **없음** — 변경 hunk 1개, 전부 `MergeConcept` 내부 |
| dependent module 수정 | **없음** |
| `go.mod` / `go.sum` 변경 | **없음** |
| 커밋 | `04d7f40` |

## H.6 기존 서술 정정

부록 D·E 및 `FORK.md` 의 P4 서술이 *"확장 키를 이월한다"* 로 **무조건적 보존처럼 읽힐 수 있었다.** 정확히는 **P4b 이전까지 그 보존은 조건부**였다 — fresh 가 확장 키를 하나도 갖지 않을 때만 성립했다. 해당 문서의 P4 항목에 이 조건을 명시하고 P4b 를 병기했다.

이 정정은 P4 의 다른 효과(=`verified`/`sources`/`usage_window`/`status`/`stale_after` 이월)에는 영향을 주지 않는다. 그 필드들은 각각 개별 조건으로 이월되며 C5 와 무관하게 동작한다(부록 D.4 end-to-end 및 boundary B5/B6/B6b 로 확인).

---

## 부록 H 추기 — F2 관련 신규 boundary candidate (수정하지 않음)

부록 C.0 의 F2 관측(충돌 evidence 처리 규칙 부재, unresolved 상태 표현 부재)은 **재도출하지 않는다.** 아래는 F2 를 정책으로 통합하는 과정에서 새로 **실측된 runtime evidence** 만 기록한 것이다.

**관측**: `MergeConcept` 은 `merged.Sources` 가 비어 있을 때만 `existing.Sources` 를 이월한다. connector 의 fresh 문서가 자체 `sources` 를 방출하면 **기존 provenance 전체가 대체**된다.

| Fixture | 마커 | sources |
|---|---|---|
| fresh 에 `sources` 없음 | 생존 | **2개 생존** (`schema-doc`, `pipeline-spec`) |
| **fresh 가 자체 `sources` 방출** | 생존 | **1개** (`conn`) — 충돌 evidence 양쪽 소실 |
| fresh 가 자체 확장키 방출 | 생존 | 2개 생존 (P4b 동작 재확인) |

**성격**: C5 와 동일한 all-or-nothing 형태이나 대상이 **SPEC 모델 필드(`Sources`)** 다. 마커는 살아남지만 **그 마커가 가리키는 충돌 evidence 가 사라진다.**

**상태**: **boundary candidate — 이번 단계에서 수정하지 않았다.** 후속 범위는 `Sources` 단독이 아니라 동일 형태를 갖는 **리스트 계열 이월 전반**으로 잡아야 한다. 상세와 정책 요구사항은 `DERIVED-REQUIREMENTS.md` §5.7–5.8.

**이번 단계 코드 변경**: 없음 (`okf-go` / `okf-enrich` / `okf-viz` / `okf-lint` 전부 무수정, fork `dirty=0` @ `1897244`).

---

## 부록 H 추기 2 — `MergeConcept` 이월 정책 범위 측정 (수정 없음)

**목적**: F2 에서 드러난 `Sources` 소실이 단일 필드 문제인지, 이월 정책 전반의 문제인지 **범위만** 확정한다. 코드 수정 없음(`fork dirty=0`).

**측정 방향**: 지금까지의 실측·게이트는 전부 *fresh 가 해당 필드를 비워 둔* 방향만 다뤘다. 이번에는 **fresh 가 값을 들고 오는 방향**을 잰다.

| 필드 | existing | fresh | 결과 | 분류 |
|---|---|---|---|---|
| **`Verified`** | `human:vavagirls` | `process:okf-sqlite` | **`process:okf-sqlite` 만 잔존, tier `human-reviewed` → `machine-confirmed`** | **ALL-OR-NOTHING silent-loss** |
| **`Sources`** | 기존 2건 | `conn` 1건 | **`conn` 만 잔존** | **ALL-OR-NOTHING silent-loss** |
| `Tags` | `curated,orders` | `fs,file` | `curated,file,fs,orders` | safe merge (union) |
| `UsageWindow` | from `2026-06-01` | from `2026-08-01` | fresh 값 | intentional fresh overwrite |
| `Status` | `stable` | `draft` | `draft` | intentional fresh overwrite |
| `StaleAfter` | `2026-12-31` | `2027-06-30` | `2027-06-30` | intentional fresh overwrite |
| `Extra` | `x_policy` | `x_conn` | 2 keys | safe merge (P4b) |
| `Description` | 큐레이션됨 | placeholder | 기존 유지 | existing wins (by design) |

> `intentional` 은 **코드가 그렇게 동작한다**는 뜻이며 **정책적으로 옳다는 뜻이 아니다.** 스칼라 lifecycle 값이 조용히 교체되는 것이 타당한지는 별도 정책 판단이며 이번 범위 밖이다.

### 정정 — P4 의 trust carry-over 는 조건부였다

부록 D·E 와 `FORK.md` 는 P4 를 *"구조 변경 재생성 시 `verified` 를 이월한다"* 로 서술했다. 정확히는 **fresh 가 `verified` 를 비워 둘 때만** 성립한다. connector 가 `verified` 를 하나라도 방출하면 기존 사람 검증이 통째로 대체되고 **trust tier 가 조용히 강등**된다 — 부록 C.1-T3 에서 확인했던 복합 훼손이 이 경로로 재현된다.

`Sources` 도 동일하다(부록 H 추기).

**이는 P4 의 다른 효과를 무효화하지 않는다.** `Tags`(합집합)·`Extra`(P4b)·`Description`(기존 우선)은 fresh 가 값을 들고 와도 안전하며, 스칼라 3종은 별개 성격이다.

### 우리 게이트의 사각지대

`boundary_test.go` 의 **B6(`TestBoundaryTrustStatePreserved`)·B5(`TestBoundaryProvenancePreserved`)는 fresh 가 해당 필드를 비워 둔 경우만 검사한다.** 반대 방향은 현재 15개 게이트 어디에도 없다. 즉 이 실패는 **게이트를 통과한 채로 발생한다.**

### Boundary candidate 범위 확정

**대상은 두 개의 리스트 값 필드 — `Verified` 와 `Sources`.** `Tags` 는 이미 합집합으로 안전하고 `Extra` 는 P4b 로 닫혔으므로 후속 범위에서 제외된다.

**단, 맵과 달리 리스트는 병합 전에 동일성 규칙이 필요하다.**
- `SourceEntry`: `Resource` 는 REQUIRED 이나 같은 리소스를 다른 관점으로 두 번 인용할 수 있고, `ID` 는 optional 이다
- `VerifiedEntry`: 같은 actor 의 재검증인지 별개 이벤트인지, 중복을 어떻게 볼지 판단이 필요하다 (F6 의 중복 누적 문제와 맞물린다)

동일성 규칙을 잘못 고르면 provenance 가 조용히 합쳐지거나 중복된다. **따라서 이번 단계에서 병합 규칙을 만들지 않았다.** 게이트 확장(fresh-non-empty 방향)도 규칙 확정 이후에 해야 한다.

**이번 단계 코드 변경: 없음.**

---
---

# 부록 I. P4c — 검증 이력 병합 (`Verified`)

> **범위.** 부록 H 추기 2 에서 확정한 candidate 두 개 중 **`Verified` 하나만** 닫는다. `Sources` / `UsageWindow` / `Status` / `StaleAfter` 는 건드리지 않았다.

## I.1 왜 이것만 오늘 닫는가

`Sources` 와 성격이 다르다. 새 정책 결정이 필요 없고 **SPEC 이 병합 규칙을 사실상 정해준다.**

- SPEC §5.2: *"Multiple entries capture **independent** checks"*
- `VerifiedEntry` 의 전부가 `by` 와 `at` 두 필드다 ⇒ **exact `(by, at)` 이 현재 포맷이 제공하는 유일한 event identity**

따라서 ownership 모델 설계가 아니라 **기존 이력 보존 + 동일 이벤트 중복 제거**라는 기계적 규칙으로 닫힌다. `Sources` 는 `Resource`(REQUIRED)/`ID`(optional) 사이에서 동일성 규칙을 **설계**해야 하므로 성격이 다르다.

## I.2 Baseline

**관측**: B6 은 fresh 가 `verified` 를 비워 둔 방향만 검사했다. 반대 방향에서 P4 는 리스트를 통째로 대체했다.

| 상태 | 결과 |
|---|---|
| existing `human:vavagirls` + fresh `process:okf-sqlite` | **`process:okf-sqlite` 만 잔존, tier `human-reviewed` → `machine-confirmed`** |

⇒ 부록 C.1-T3 의 복합 훼손이 이 경로로 재현된다.

**현재 위험 등급 — latent / owner-controlled (실측)**: upstream connector **12개 전부와 `okf-mcp` 가 `Verified`·`Sources` 를 방출하지 않는다**(소스 전수 검색). 즉 오늘 터지는 결함이 아니라 **우리 producer 가 `verified` 를 방출하기 시작하면 현실화**되는 잠복 경계다.

**baseline 게이트 결과** (P4c 적용 전, `TestBoundaryVerifiedHistoryMerge` 4개 서브테스트)

| 서브테스트 | 결과 |
|---|---|
| 기존 human 서명이 fresh 기계 항목과 함께 생존 | **FAIL** |
| 동일 `(by, at)` 중복 제거 | PASS (wholesale 치환이 우연히 1건을 냈다) |
| 같은 actor 의 다른 시각 = 별개 이벤트 | **FAIL** |
| 반복 호출 결정성 | **FAIL** |

## I.3 P4c — merge rule

`Verified = dedup(existing + fresh)` — **existing 먼저, 그다음 아직 없는 fresh 항목**, 동일 `(by, at)` 제거.

- **정렬하지 않고 안정 연결(stable concatenation)** 을 쓴다. `at` 은 optional 이라 부재 시각을 정렬하는 것은 임의적이기 때문이다.
- `(by, at)` 이 같고 의미가 다른 경우는 **상상하지 않는다.** 현재 SPEC 기준 최소 규칙까지만이며, 추후 SPEC 이 verification entry 에 의미 필드를 추가하면 재검토한다.

## I.4 Patched result

`TestBoundaryVerifiedHistoryMerge` **4/4 PASS**. 특히 human + process 가 함께 남을 때 trust tier 가 `human-reviewed` 로 유지됨을 확인했다.

## I.5 F6 — 범위를 한정한 해소

F6 은 *"`verified` append 에 중복 제거 규칙 없음"* 이었다. 실측:

| 경로 | 결과 |
|---|---|
| 병합 없이 직접 3회 append | **3건 유지** |
| 이후 구조 변경 재생성 1회 통과 | **1건으로 수렴** |

⇒ **F6 = REMEDIATED (merge boundary 한정).** 중복 누적은 병합 경계에서 자가 치유되며, 주 생성 경로였던 §4c 자기 append 는 부록 F 에서 이미 제거되었다. 다만 **병합 사이에 직접 기록된 중복은 다음 재생성까지 남는다.** `CLOSED` 로 표기하지 않는다.

## I.6 Regression

| 항목 | 결과 |
|---|---|
| **Producer boundary gate** | **15 → 16** 함수, **16/16 PASS** |
| upstream `okf-go` tests | PASS |
| 워크스페이스 16개 모듈 | **16/16 PASS** |
| F8 trust policy gate | PASS |

**게이트 계수**: Producer boundary gate **16** / Trust policy enforcement gate **+1**. 합산하지 않는다.

## I.7 Cost

| 항목 | 값 |
|---|---|
| 수정 파일 | **2** (`okf-go/hash.go`, `okf-go/boundary_test.go`) |
| 소스 변경 | `hash.go` **+34 / −3** (hunk 2개: 이월 1줄 교체 + `mergeVerified` 헬퍼 신규) |
| `okf-go` 외 변경 | **없음** |
| dependent module / `go.mod` / `go.sum` | **변경 없음** |
| 커밋 | `ed4fbfb` |

## I.8 남은 것 (오늘 닫지 않음)

- **`Sources` all-or-nothing** — 항목 동일성 규칙 설계가 선행되어야 한다. boundary candidate 유지
- **`UsageWindow` / `Status` / `StaleAfter`** — fresh 우선이 타당한지는 ownership semantics 판단. 오늘 작업과 종류가 다르다
- **게이트 사각지대** — B5(`Sources`) 의 fresh-non-empty 방향은 여전히 게이트에 없다. `Verified` 방향만 B15 로 편입되었다
