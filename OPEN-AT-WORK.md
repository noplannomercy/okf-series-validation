# 회사 PC에서 여는 법

2026-08-29 작업분. 이 폴더 하나만 옮기면 된다. **GitHub 없이도 전부 복원된다.**

## 들어 있는 것

| 파일 | 내용 |
|---|---|
| `okf-skills-fork.bundle` | fork 저장소 전체 (커밋 9개, 파일 249개). **0.6 MB** |
| `OKF-RE-VALIDATION.md` | 검증 리포트 1294줄 — 증거와 asset별 판정 |
| `DERIVED-REQUIREMENTS.md` | 파생 요구사항 초안 (리포트 밖, 미확정) |
| `okf-go-fork.patch` | upstream 대비 okf-go 소스 diff (참고용) |

> 번들이 0.6 MB 인 이유: fork 자체 내용만 담았다. upstream 이력 팩(57 MB)은 필요할 때 회사에서 다시 받으면 된다.

## 1. 저장소 복원

```bash
git clone okf-skills-fork.bundle okf-skills-fork
cd okf-skills-fork
```

upstream 추적이 필요하면 (`git diff upstream/master` 용):

```bash
git remote add upstream https://github.com/xSAVIKx/okf-skills.git
git fetch upstream
```

## 2. Go 툴체인

`go.work` 가 **Go 1.26 이상**을 요구한다. 회사 PC에 없으면 이 PC에서 쓴 방식 그대로:

```bash
# 포터블 zip — 시스템 PATH·레지스트리 안 건드림
curl -sL -o go.zip https://go.dev/dl/go1.26.7.windows-amd64.zip
# 압축 해제 후 그 세션에서만:
export GOROOT=<풀린경로>/go
export GOPATH=<작업폴더>/gopath
export PATH=$GOROOT/bin:$PATH
export GOTOOLCHAIN=local
```

## 3. 게이트 확인 — 이게 정상 동작의 기준

```bash
# Producer boundary gate: 16개 함수 전부 PASS 여야 함
cd okf-go && go test -run TestBoundary ./...

# Trust policy enforcement gate (opt-in)
cd ../skills/okf-lint && go test -run TestPolicy ./...

# 전체 16개 모듈
# (workspace 루트에서 각 모듈별로 go test ./...)
```

**게이트가 깨지면 fork 의 존재 이유가 회귀한 것이다.** 상류를 병합했을 때 특히.

## 4. 월요일 첫 작업 — 30초

`okf-oracle` 에 **`go.mod` 가 있는지만** 본다 (read-only).

- **있으면** → wiring 문제. `go.work` 에 fork 경로 추가로 끝
- **없으면** → 계약 이식 문제. `DERIVED-REQUIREMENTS.md` 의 항목들을 해당 구현 언어로 옮겨야 함

같이 볼 것: `go.work` / `replace` / import path / 빌드가 실제로 선택하는 `okf-go`.

이 결과가 남은 작업 범위를 정한다. 그전까지 `okf-oracle` 관련 상태는 **HOLD-ENV** 다.

## 5. 리포트 읽는 순서

`OKF-RE-VALIDATION.md` 는 §0–§9 가 1차 검증이고 **부록 A–I 가 그 위에 얹은 재검토다.**
**부록이 본문 판정을 갱신한 경우 부록이 최신이다.** 헤더에 정리해 뒀다.

바쁘면 이것만:
- **§9 + 부록 A.4** — asset 별 최종 판정
- **부록 I.8** — 남은 것
- `okf-skills-fork/FORK.md` — 패치 P1~P5, P4b, P4c 와 각각의 SPEC 근거

## GitHub 에 올리는 건?

필요 없다. 다만 올리고 싶다면 확인할 것:

- 회사 PC 에서 **개인 GitHub 로 push 가 정책상 허용되는지**
- 커밋 author 에 개인 이메일이 박혀 있다 (`vavagirls@gmail.com`)
- 리포트에 **내부 프로젝트 명칭**(G-Core, okf-oracle)이 들어 있다 → 공개 저장소는 부적절, private 만

코드 자체는 Apache-2.0 공개 코드 + 우리 패치라 라이선스 문제는 없다.
