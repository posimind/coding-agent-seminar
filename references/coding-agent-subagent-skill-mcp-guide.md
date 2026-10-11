# Subagent · Skill · MCP 비교와 선택 가이드

> 작성일: 2026-10-11 · 예시는 Claude Code 기준

## 1. 개요

Subagent, Skill, MCP 중 무엇을 골라도 같은 목표를 구현할 수 있다. 셋 다 결국 **모델 컨텍스트에 텍스트를 넣고, 실행 가능한 능력을 붙이는 방법**이기 때문이다. 그러나 각자 최적화된 지점이 달라서, 효율을 기준으로 보면 역할이 나뉜다.

| 메커니즘 | 비유 | 답하는 질문 |
| --- | --- | --- |
| Subagent | 별도의 두뇌 | **누가** 생각하는가 (격리된 컨텍스트에서의 위임) |
| Skill | 매뉴얼 | **어떻게** 하는가 (필요할 때 펼치는 절차 지식) |
| MCP | 손발 | **무엇에** 닿는가 (시스템 경계 밖으로의 접근) |

---

## 2. 각각의 본질

| 항목 | Subagent | Skill | MCP |
| --- | --- | --- | --- |
| 정체 | 독립 컨텍스트 윈도우를 가진 별도 에이전트 인스턴스 | 온디맨드로 로드되는 지침 + 번들 리소스(스크립트, 참조 문서) | 외부 능력을 표준 프로토콜로 노출하는 서버 |
| 메인 컨텍스트에 들어가는 것 | 최종 결과 요약만 | 평소엔 name·description만, 필요 시 본문. 스크립트는 코드가 아닌 출력만 | tool 스키마 + 호출 결과 |
| 실행 주체 | 새 LLM 루프 | 호출한 에이전트 자신 (그 셸과 권한) | 별도 프로세스(stdio) 또는 원격 서버(HTTP) |
| 고유 능력 | 컨텍스트 격리, 병렬화, 모델·권한 분리 | 지식의 재사용과 지연 로딩 | 자격증명 격리, 강제 가드레일, 상태 유지 |
| 트리거 | 메인 에이전트가 위임 | 모델이 description을 보고 자율 로드 | tool은 모델, prompt는 사용자, resource는 애플리케이션 |
| 비용 성격 | 호출마다 토큰·지연 (콜드 스타트, 맥락 재구성) | 거의 없음 | 서버 운영 (배포, 인증, 버전 관리) |

### 참고: MCP primitive별 제어 주체

MCP의 prompt도 지침을 주입할 수 있지만, 모델이 스스로 꺼내 쓰는 구조가 아니다.

| Primitive | 제어 주체 | 의미 |
| --- | --- | --- |
| Tools | 모델 | 모델이 스스로 판단해 호출 |
| Resources | 애플리케이션 | 클라이언트가 컨텍스트에 첨부 |
| Prompts | 사용자 | 보통 `/mcp__server__prompt` 같은 슬래시 커맨드로 사용자가 선택 |

서버 초기화 시 `instructions` 필드로 상시 지침을 줄 수도 있지만, 연결된 순간부터 항상 로드되므로 Skill의 온디맨드 로딩과는 성격이 반대다.

---

## 3. 같은 목표를 세 방식으로 각각 구현하기

**목표: "PR을 가져와 체크리스트대로 리뷰하고 코멘트를 남긴다"**

### (A) Subagent만으로

시스템 프롬프트에 체크리스트를 넣고, Bash로 `gh`를 쓰게 한다.

```markdown
# .claude/agents/pr-reviewer.md
---
name: pr-reviewer
description: PR을 리뷰하고 코멘트를 남길 때 사용
tools: Bash, Read, Grep
---
너는 코드 리뷰어다. `gh pr diff <번호>`로 diff를 가져와
1) 메모리 안전성 2) 에러 처리 3) 네이밍 규칙 순으로 검토하고
`gh pr comment`로 결과를 남겨라.
```

### (B) Skill만으로

체크리스트는 본문에, 외부 연결은 스크립트로.

```markdown
# skills/pr-review/SKILL.md
---
name: pr-review
description: PR을 체크리스트 기준으로 리뷰할 때 사용
---
1. `bash scripts/fetch_diff.sh <번호>` 실행
2. references/checklist.md 기준으로 검토
3. `bash scripts/post_comment.sh <번호> <파일>`로 게시
```

```bash
# skills/pr-review/scripts/fetch_diff.sh
#!/usr/bin/env bash
# 에이전트의 셸에서 실행됨 → 토큰이 에이전트 환경에 있어야 함
curl -s -H "Authorization: Bearer ${BITBUCKET_TOKEN}" \
  "${BITBUCKET_URL}/rest/api/1.0/projects/${PROJ}/repos/${REPO}/pull-requests/$1/diff"
```

### (C) MCP만으로

도구 + prompt 템플릿. 극단적으로는 서버 안에서 LLM까지 호출할 수 있다.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("review")

@mcp.tool()
def get_pr_diff(pr: int) -> str:
    """Get the diff of a pull request."""
    ...

@mcp.tool()
def post_comment(pr: int, body: str) -> str:
    """Post a review comment on a pull request."""
    ...

@mcp.prompt()
def review(pr: int) -> str:
    return f"PR {pr}을 1) 메모리 안전성 2) 에러 처리 3) 네이밍 순으로 리뷰하라"

# 혹은 서버가 sampling으로 클라이언트 LLM을 불러 리뷰까지 수행 → 사실상 서브에이전트
```

**셋 다 동작한다.** 어떤 메커니즘이든 "지침 텍스트 + 실행 능력"을 조합할 수 있기 때문이다. 이것이 세 개념이 모호하게 느껴지는 이유다.

---

## 4. 남의 역할을 할 때의 비효율

구현은 가능하지만, 본업이 아닌 칸에서는 대가를 치른다.

| 시키는 일 → | 격리된 작업 수행 | 절차 지식 제공 | 외부 접근 |
| --- | --- | --- | --- |
| **Subagent** | ◎ 본업 | △ 지식이 그 에이전트에 갇힘. 메인이나 다른 서브에이전트가 재사용 불가, 쓸 때마다 스폰 비용 | ✕ 스스로 접근 능력이 없음. 결국 Bash나 MCP를 빌려 씀 |
| **Skill** | △ 메인 컨텍스트에서 실행되어 중간 산출물이 쌓이고 오염됨. 병렬화·모델 분리 불가 | ◎ 본업 | △ 토큰이 에이전트 환경에 노출, 규칙은 "권고"일 뿐 강제 불가, 셸 필요 |
| **MCP** | △ sampling으로 가능하나 클라이언트 지원 편차가 크고, 추론 과정이 서버 안에 숨어 디버깅이 어려움 | △ prompt는 사용자 호출 구조, instructions는 항상 로드. 지침 수정에 서버 재배포 필요 | ◎ 본업 |

대가는 크게 세 가지로 요약된다.

- **컨텍스트 오염**: Skill로 격리 작업을 하면 중간 결과가 메인에 쌓인다.
- **지식 고립**: Subagent나 MCP에 매뉴얼을 넣으면 다른 곳에서 재사용하기 어렵다.
- **보안 노출**: Skill로 외부 연결을 하면 자격증명과 가드레일을 에이전트에게 맡기게 된다.

### 예: Skill 규칙은 권고, MCP 규칙은 강제

```markdown
<!-- SKILL.md: 모델이 무시하거나 인젝션으로 우회될 수 있음 -->
- protected 저장소의 PR은 절대 머지하지 말 것
```

```python
# MCP 서버: 모델이 우회할 수 없음
@mcp.tool()
def merge_pr(repo: str, pr_id: int) -> str:
    if repo in PROTECTED_REPOS:
        raise ValueError("protected repo: merge는 사람만 가능")
    if not ci_passed(repo, pr_id):
        raise ValueError("CI 미통과")
    return bitbucket.merge(repo, pr_id)
```

---

## 5. 선택 가이드

구현하려는 기능을 **"누가 / 어떻게 / 무엇에"의 세 관심사로 쪼갠 뒤**, 각 관심사를 해당 메커니즘에 배정한다.

### Subagent로 빼야 하는 신호

- [ ] 중간 산출물이 커서 메인 컨텍스트를 더럽힌다 (대량 로그 분석, 코드베이스 전체 탐색)
- [ ] 서로 독립적인 작업을 병렬로 돌릴 수 있다
- [ ] 다른 모델이나 더 좁은 권한으로 돌리고 싶다 (읽기 전용 리뷰어, 저렴한 모델의 탐색기)
- [ ] 독립적 판단이 필요하다 (작성자와 분리된 검증자, 적대적 리뷰)

### Skill로 만들어야 하는 신호

- [ ] 반복되는 절차, 사내 규칙, 판단 기준이다
- [ ] 메인 에이전트와 여러 서브에이전트가 공통으로 참조해야 한다
- [ ] 자주 쓰지 않아 상시 로드하기엔 컨텍스트가 아깝다
- [ ] 내용이 자주 바뀌어 파일 수정만으로 갱신하고 싶다

### MCP로 만들어야 하는 신호

- [ ] 자격증명을 에이전트에게 보이면 안 된다
- [ ] 쓰기·삭제 작업에 우회 불가능한 가드레일이 필요하다
- [ ] 세션·커넥션 같은 상태가 필요하다
- [ ] 셸 없는 클라이언트에서도 써야 하거나, 여러 팀이 같은 연결을 공유한다

### 기본값 순서

1. **Skill에서 시작한다.** 비용이 가장 싸고, 매뉴얼은 어차피 Skill에 둔다. 외부 연결도 초기엔 스크립트로 충분하다.
2. **접근 문제가 생기면 MCP를 추가한다.** 자격증명 격리, 강제 가드레일, 상태, 공유 요구가 생기는 시점.
3. **컨텍스트·병렬·독립성 문제가 생기면 Subagent로 분리한다.**

### 승격을 쉽게 만드는 구조

연결 로직을 처음부터 코어 라이브러리로 분리하면, Skill 스크립트에서 MCP로 옮기는 비용이 거의 없다.

```
pr-review/
├── core/review_ops.py   # 실제 로직 (순수 함수)
├── cli.py               # 1단계: Skill 스크립트가 호출하는 얇은 래퍼
├── server.py            # 2단계: MCP 래퍼 (필요해지면 추가)
└── skill/SKILL.md       # 매뉴얼
```

```python
# core/review_ops.py
def fetch_diff(client, pr: int) -> str:
    return client.get_diff(pr)

# cli.py — 1단계
import sys
from core.review_ops import fetch_diff
print(fetch_diff(make_client(), int(sys.argv[1])))

# server.py — 2단계
from mcp.server.fastmcp import FastMCP
from core.review_ops import fetch_diff
mcp = FastMCP("bitbucket")

@mcp.tool()
def get_pr_diff(pr: int) -> str:
    """Get the diff of a pull request."""
    return fetch_diff(make_client(), pr)
```

승격 시 SKILL.md는 한 줄만 바뀐다.

```diff
- 1. `python cli.py <번호>` 로 diff 조회
+ 1. `mcp__bitbucket__get_pr_diff(pr=<번호>)` 로 diff 조회
  2. references/checklist.md 기준으로 검토 ...
```

---

## 6. 역할대로 조합한 예시

같은 PR 리뷰를 역할대로 나누면 다음과 같다.

**두뇌: 격리 + 읽기 전용 + 저렴한 모델**

```markdown
# .claude/agents/pr-reviewer.md
---
name: pr-reviewer
description: PR diff를 독립적으로 리뷰할 때 사용
tools: Read, Grep, Glob, mcp__bitbucket__get_pr_diff
model: sonnet
skills: review-checklist
---
diff를 검토하고 지적사항을 구조화해 반환하라. 직접 게시하지 마라.
```

**매뉴얼: 메인/서브 모두 재사용**

```markdown
# skills/review-checklist/SKILL.md
---
name: review-checklist
description: 사내 C++/Rust 코드 리뷰 기준이 필요할 때 사용
---
1. 메모리 안전성: raw pointer 소유권, unsafe 블록 사유 주석 ...
2. 에러 처리: Result 무시 금지, errno 확인 ...
3. 네이밍: 모듈 접두어 규칙 ...
```

**손발: 토큰 격리 + 강제 가드레일**

```python
# bitbucket MCP 서버
@mcp.tool()
def get_pr_diff(pr: int) -> str:
    """Get the diff of a pull request."""
    ...

@mcp.tool()
def post_comment(pr: int, body: str) -> str:
    """Post a review comment on a pull request."""
    if len(body) > 10_000:
        raise ValueError("코멘트 과다")
    ...
```

**실행 흐름**

1. 메인 에이전트가 `pr-reviewer` 서브에이전트에 리뷰를 위임한다.
2. 서브에이전트는 `review-checklist` Skill을 참고하고, MCP로 diff만 읽는다.
3. 서브에이전트는 지적사항을 요약해 메인에 반환한다. 메인 컨텍스트에는 diff 전체가 아닌 요약만 남는다.
4. 메인 에이전트가 사용자 확인 후 `post_comment`로 게시한다.

서브에이전트에는 쓰기 도구를 아예 주지 않았으므로 권한 분리도 자연스럽게 이루어진다.

---

## 7. 요약

- **"누가 생각하나"가 문제면 Subagent, "어떻게 하나"가 문제면 Skill, "무엇에 닿나"가 문제면 MCP.**
- 셋 다 무엇이든 구현할 수는 있지만, 본업이 아닌 칸에서는 컨텍스트 오염, 지식 고립, 보안 노출 중 하나를 대가로 치른다.
- 기본값은 Skill. 접근 문제가 생기면 MCP, 컨텍스트·병렬·독립성 문제가 생기면 Subagent를 더한다.
