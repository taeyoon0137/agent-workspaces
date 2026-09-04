# `~/.agents` — 공용 에이전트 설정 저장소

> **이 문서는 인간을 위한 안내입니다.**
> 
> 에이전트는 본 문서를 읽으라고 직접적으로 지시받지 않았다면 이 문서를 읽지 않습니다. 다만 그래도 종종 읽을 수 있어, 컨텍스트 절약을 위해 최소한으로만 기재합니다.

에이전트가 컨텍스트 압축이나 세션 재시작, 드리프트 등으로 인해 맥락을 누락하거나 지시 사항을 이행하지 못하지 않도록 참조할 수 있는 전역 저장소입니다.

가벼운 형태의 명령어 구성을 통해 언제 어디서든 불러올 수 있도록 제공됩니다.

## 시작하기

아래 명령어를 에이전트에게 실행 요청하세요. 단, `~/` 경로에 설치하므로 주의가 필요합니다.

```plaintext
아래 주소의 지침을 따라 해당 레포지토리를 설치할 것.
지침을 임의로 해석하거나 판단하지 말 것.
https://raw.githubusercontent.com/taeyoon0137/agent-workspaces/main/INSTALLATION.md
```

이후 아래 명령어를 통해 권한 부여 및 스킬 설치가 필요합니다.


```bash
# ctx 실행 권한 (동기화·복사 과정에서 빠질 수 있음)
chmod +x ~/.agents/bin/ctx.sh

# 네이티브 플러그인 등록 — 슬래시 명령(/i-have-adhd, /ponytail-review 등)과 훅용.
# 지침 자체는 AGENTS.md와 skills/에 내장되어 있어 등록 없이도 적용됨.
claude plugin marketplace add ayghri/i-have-adhd && claude plugin install i-have-adhd@i-have-adhd
claude plugin marketplace add DietrichGebert/ponytail && claude plugin install ponytail@ponytail
codex plugin marketplace add ayghri/i-have-adhd --ref main && codex plugin add i-have-adhd@i-have-adhd
codex plugin marketplace add DietrichGebert/ponytail && codex plugin add ponytail@ponytail
```


## 구성

```plaintext
.agents/
├── bin/ctx.sh                    워크스페이스 레코드 관리 도구(POSIX sh)
├── skills/                       에이전트들을 위해 공용으로 로드되는 스킬
│   ├── find-skills/              설치된 스킬 탐색을 위한 공용 스킬
│   ├── workspace-context/        본 워크스페이스 탐색을 위한 명령어셋 스킬
│   └── workspace-skill-create/   워크스페이스 로컬 스킬 생성·갱신 절차
├── workspaces/<id>/              저장소별 외부 컨텍스트. <id> = <폴더명>-<경로 sha256 앞 16자>
│   ├── artifacts/                레코드가 아닌 에이전트 산출물
│   ├── records/                  레코드 1건 = 파일 1개. D-결정 C-체크포인트 E-증거 B-블로커 F-사실 H-이력
│   ├── skills/<name>/            이 워크스페이스에서만 유효한 로컬 스킬 (CONTEXT.md ## Skills로 발견)
│   ├── CONTEXT.md                현재 상태만 (고정 8섹션, 3,000자/40줄 캡, 교체만 허용)
│   ├── FACTS.md                  현재 사실 요약 (records/F-*.md에서 ctx index가 생성)
│   └── INDEX.md                  records/ 카탈로그 (ctx index가 생성, 수동 편집 금지)
└── AGENTS.md                     모든 에이전트 공용 정책
```

## 컨텍스트 레코드 `ctx`

각 워크스페이스에서의 방대한 정보들을 에이전트가 직접 읽고 탐색하길 방지하고자, 간단한 형태의 명령어 구성을 만들어 제공하였습니다.

> `ctx`로 호출하려면 `export PATH="$HOME/.agents/bin:$PATH"`와 `alias ctx=ctx.sh`를 셸 설정에 추가합니다. 캡은 `CTX_MAX_CHARS`, `CTX_MAX_LINES` 환경변수로 조정 가능합니다.


```bash
ctx status              # 캡 준수 여부, 레코드 수, 레거시 레이아웃 감지
ctx check               # 스키마·인덱스·링크 검증 (에이전트가 턴 종료 전 실행)
ctx find <정규식>       # INDEX.md 검색 → ctx show <id>
ctx add decision "제목" -t 태그 -r "읽는 시점" [-s 대체할-id]   # 본문은 stdin
ctx set <id> status resolved
ctx help
```
