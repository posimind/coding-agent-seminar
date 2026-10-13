# Part 1. 코딩 에이전트 도구 ③ 확장 부품과 셋업 (5\~7장)

## 5장. 확장 부품 ①: 지침 · Skill · Hook

- **핵심 질문**: 에이전트에게 우리 프로젝트의 규칙을 어떻게 알려주고, 어떻게 지키게 하나요
- **도달 목표**: 같은 규칙을 지침·Skill·Hook 중 어디에 둘지 판단합니다

4장은 "비용을 통제하면서 에이전트를 내 프로젝트에 맞추는 부품에는 무엇이 있는가"로 끝났습니다. 부품은 여섯 가지입니다. 지침, Skill, Hook, MCP, Subagent, Plugin입니다. 모두 2.1절에서 본 사용자 하네스에 속하므로 우리가 만들고 우리가 고칩니다. 이 장은 프로젝트 안에서 쓰는 셋을 다룹니다. 규칙을 알려 주는 지침과 Skill, 규칙을 지키게 하는 Hook입니다. 프로젝트 밖으로 나가는 나머지 셋은 6장에서 봅니다.

세 부품을 가르는 질문은 둘입니다. **언제 컨텍스트에 실리는가**, 그리고 **요청인가 강제인가**. 앞의 질문이 비용을 정하고(4장), 뒤의 질문이 얼마나 믿을 수 있는지를 정합니다(2.3절).

### 5.1 부품 지도

부품마다 컨텍스트에 실리는 시점과 양이 다릅니다. 아래 표가 5\~7장의 지도입니다. 마지막 열은 2.3절의 가이드·센서 2×2에서 그 부품이 놓이는 칸입니다.

| 부품 | 로드 시점 | 컨텍스트에 들어가는 것 | 비용 | 2.3절의 칸 |
| --- | --- | --- | --- | --- |
| 지침 파일 | 세션 시작 | 전문 | 매 호출에 실립니다 | 추론적 가이드 |
| 조건부 룰, 하위 디렉터리 지침 | 해당 파일을 다룰 때 | 전문 | 로드된 뒤부터 실립니다 | 추론적 가이드 |
| Skill | 시작 때 설명, 쓸 때 본문 | 설명 한 줄. 쓰면 본문 | 낮습니다. 설명만 매 호출에 실립니다 | 추론적 가이드. 리뷰 체크리스트로 쓰면 추론적 센서 |
| Hook | 이벤트가 일어날 때 | 없습니다. 훅이 돌려준 출력만 들어갑니다 | 0. 돌려준 출력만큼만 듭니다 | 계산적 센서를 루프 안에서 돌리는 자리 |
| MCP 서버 | 세션 시작 | tool search가 켜져 있으면 도구 이름과 서버 설명.<br>꺼져 있으면 모든 도구의 전체 스키마 | 켜져 있으면 쓰기 전까지 낮습니다.<br>꺼져 있으면 매 호출에 전부 실립니다 | 칸에 들지 않습니다. 가이드와 센서가 쓸 자료와 수단을 댑니다 |
| Subagent | 호출될 때 | 메인 대화에는 결과 요약만 | 메인과 격리됩니다. 총 토큰은 늘 수 있습니다 | 리뷰용이면 추론적 센서 |
| Plugin | 켜 둔 동안 | 담긴 부품이 각자의 규칙대로 | 담긴 부품의 합 | 칸이 아니라 배포 단위 |

(출처: [Claude Code 문서, Extend Claude Code](https://code.claude.com/docs/en/features-overview)의 "Context cost by feature" 표, 2026-10 확인. 마지막 열과 MCP 행의 조건은 이 세미나가 덧붙였습니다.)

MCP 행은 조건을 붙여 읽습니다. 공식 문서의 표는 MCP를 "Tool names; full schemas on demand"(도구 이름만 싣고 전체 스키마는 필요할 때 싣는다)로 적습니다. 이것은 tool search가 켜져 있을 때의 동작입니다. `ANTHROPIC_BASE_URL`을 바꿔 쓰는 사내 환경에서는 tool search가 기본으로 꺼지고, 그러면 연결한 서버의 전체 스키마가 매 호출에 실립니다(4.4절, 4.8절).

이 표의 비용 열은 1.5절의 `/context` 화면에서 실측으로 확인됩니다. `Memory files`, `Skills`, `MCP tools`, `Custom agents` 행이 각각 지침, Skill, MCP, Subagent의 몫입니다. 표에서 읽을 것은 두 가지입니다.

- **위의 세 행은 모두 모델이 읽고 해석하는 글입니다.** 따라 주기를 기대하는 요청이라는 뜻입니다. 모델의 해석을 거치지 않고 실행되는 것은 Hook뿐입니다(5.5절).
- **비용이 가장 확실하게 붙는 것은 맨 위 행입니다.** 지침 파일에 넣은 한 문단은 호출 수만큼 곱해집니다(1.4절). 그래서 5.2절은 지침을 짧게 쓰는 법을, 5.3절은 내용을 아래 행으로 내리는 법을 다룹니다.

### 5.2 지침 파일

지침 파일은 에이전트가 세션을 시작할 때마다 읽는 프로젝트 상수입니다. 새로 온 동료에게 주는 온보딩 문서에 해당합니다. 이름은 둘입니다. `AGENTS.md`는 여러 코딩 에이전트가 함께 읽는 공개 형식이고, `CLAUDE.md`는 Claude Code의 형식입니다.

Claude Code가 둘 중 무엇을 읽는지는 3.5절에서 본 규칙을 따릅니다. 기본값에서는 `CLAUDE.md`를 읽고, `CLAUDE.md`가 없을 때만 `AGENTS.md`를 읽습니다. 이 규칙에는 조건이 붙습니다(출처: [Claude Code 문서, Memory](https://code.claude.com/docs/en/memory#agents-md), 2026-10 확인).

- **버전.** `AGENTS.md`를 직접 읽는 기능은 v2.1.277부터입니다. Amazon Bedrock이나 텔레메트리를 끈 세션처럼 일부 세션은 v2.1.281 이전에는 `CLAUDE.md`만 읽었습니다. 사내 게이트웨이 연결이 거기에 드는지는 문서로 확정되지 않으므로 v2.1.281 이상을 기준으로 삼습니다(3.5절).
- **`CLAUDE.local.md`도 "있음"으로 칩니다.** 작업 디렉터리나 그 위에 `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md` 중 하나라도 있으면 기본값에서는 `AGENTS.md`를 읽지 않습니다. 개인 메모용으로 `CLAUDE.local.md`를 만드는 순간 팀의 `AGENTS.md`가 빠집니다.
- **둘 다 읽게 하려면 설정을 바꿉니다.** 세션에서 `/config`를 열어 Project instructions 항목을 `claude-md-and-agents-md`로 둡니다.

> **발표 전 확인**: 사내 환경에서 `AGENTS.md`만 둔 저장소로 세션을 열어, 시작할 때 `no CLAUDE.md found; AGENTS.md loaded: …` 줄이 나오는지와 `/config`에 Project instructions 항목이 보이는지 확인합니다. 항목이 보이지 않으면 그 세션은 `AGENTS.md`를 직접 읽지 못하므로 5.3절 끝의 `@AGENTS.md` 방식을 씁니다.

지침은 흔히 "3계층"으로 소개됩니다. 이때 세는 것은 개인 전역, 프로젝트, 하위 디렉터리의 세 자리입니다. 공식 문서의 위치 표는 네 줄(Managed policy, User, Project, Local)이고, 하위 디렉터리 지침과 `.claude/rules/`는 그 표 밖에서 따로 다룹니다. 한 표로 모으면 다음과 같습니다.

| 범위 | 위치 | 무엇을 두는가 | 로드 |
| --- | --- | --- | --- |
| 조직 | Linux는 `/etc/claude-code/CLAUDE.md` | IT 부서가 배포하는 전사 규칙 | 세션 시작 |
| 사용자 | `~/.claude/CLAUDE.md` | 개인 습관. 모든 프로젝트에 적용됩니다 | 세션 시작 |
| 프로젝트 | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md`. 위 조건에서는 `./AGENTS.md` | 팀이 공유하는 빌드 명령, 규약, 함정 | 세션 시작 |
| 로컬 | `./CLAUDE.local.md` | 이 프로젝트에서 나만 쓰는 메모. `.gitignore`에 넣습니다 | 세션 시작 |
| 하위 디렉터리, 조건부 룰 | `src/can/CLAUDE.md`, `.claude/rules/*.md` | 일부 경로에만 해당하는 규칙 | 필요할 때 (5.3절) |

계층은 서로 덮어쓰지 않고 **이어 붙습니다.** 범위가 넓은 것부터 실리고, 실행한 위치에 가까운 지침이 마지막에 읽힙니다. 두 파일이 서로 다른 말을 하면 어느 쪽이 이긴다는 규칙이 없으므로, 충돌하지 않게 관리하는 것이 쓰는 사람의 일입니다.

#### 작성 예시

차체 제어기 펌웨어 프로젝트의 지침을 예로 듭니다. 프로젝트 이름, 경로, 명령은 모두 가상입니다.

```markdown
# 차체 제어기(BCM) 펌웨어

## 빌드와 테스트
- 호스트 빌드는 `./build.sh target=host`, 단위 테스트는 `./run_tests.sh unit`이다. 변경한 뒤에는 통과를 확인하고 보고한다
- 타깃 빌드(`target=soc-a`)는 20분이 걸린다. 사람이 요청할 때만 돌린다

## 이 코드베이스의 함정
- `src/can/`의 `*_Isr` 함수는 `startup/vectors.c`의 벡터 테이블에 함수 포인터로 등록된다. 호출처를 검색해도 나오지 않는다
- ISR 안에서는 블로킹 호출을 쓰지 않는다. `printf`, `malloc`, 뮤텍스 대기가 여기에 든다. 로그는 `LOG_ISR()`로 남긴다
- `gen/` 아래는 ARXML에서 생성한 코드다. 직접 고치지 않고 원본과 생성 스크립트를 고친다

## 판단 기준
- 새 코드는 같은 디렉터리의 기존 코드처럼 읽히게 쓴다. 이름 규칙과 오류 처리 방식을 주변에서 따온다
- MISRA C:2012를 따른다. 규칙을 어겨야 하면 멈추고 이유를 보고한다. 승인된 예외 목록은 `docs/misra-deviations.md`에 있다
- 공개 헤더(`include/`)나 CAN 메시지 정의를 바꿔야 하면 구현하기 전에 계획을 먼저 보여 준다
```

이 예시에 담긴 것은 세 종류입니다. 모델이 추측할 수 없는 **명령**, 코드를 읽어도 드러나지 않는 **함정**, 규칙 목록 대신 쓴 **판단 기준**입니다. 함정의 첫 줄은 9장에서 다시 나옵니다. 검색에 걸리지 않는 연결은 사람이 미리 알려 주지 않으면 에이전트가 볼 방법이 없습니다.

#### 좋은 지침과 나쁜 지침, 200줄 규율

2.5절에서 본 대로, 촘촘한 규칙과 반복 강조는 최신 모델에서 과잉 제약이 됩니다. 지침도 같습니다. 규칙을 나열하고 강조를 되풀이하는 대신 **판단 기준과 프로젝트 고유의 함정**을 적습니다.

| 나쁜 지침 | 좋은 지침 | 이유 |
| --- | --- | --- |
| "깨끗하고 좋은 코드를 작성한다" | (지웁니다) | 행동을 바꾸지 않는 문장은 비용만 듭니다 |
| "테스트를 잘 작성한다" | "`./run_tests.sh unit` 통과를 확인한 뒤 완료를 보고한다" | 실행하고 확인할 수 있어야 지켜집니다 |
| "주석을 절대 달지 않는다" | "주변 코드처럼 읽히게 쓴다" | 규칙은 예외 상황에서 틀린 지시가 됩니다. 판단 기준은 어디서도 틀리지 않습니다 |
| 같은 금지를 세 군데에 대문자로 반복합니다 | 한 번만 적습니다. 반드시 지켜야 하면 Hook으로 내립니다 | 강조한 줄이 많으면 어느 것도 눈에 띄지 않고, 중복은 충돌이 됩니다 |

줄마다 던질 질문은 하나입니다. Claude Code 문서의 표현으로는 "Would removing this cause Claude to make mistakes?"(이 줄을 지우면 Claude가 실수하게 되는가)입니다. 아니라면 지웁니다. 코드를 읽으면 알 수 있는 것도 적지 않습니다(출처: [Claude Code 문서, Best practices](https://code.claude.com/docs/en/best-practices#write-an-effective-claude-md), 2026-10 확인). 분량에는 공식 권고가 있습니다.

> "target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."
> (`CLAUDE.md` 파일 하나는 200줄 아래를 목표로 한다. 파일이 길수록 컨텍스트를 더 쓰고 지침을 따르는 정도가 떨어진다.)
> 출처: [Claude Code 문서, Memory](https://code.claude.com/docs/en/memory#write-effective-instructions), 2026-10 확인

200줄은 넘으면 잘리는 한도가 아니라 권고입니다. 넘어도 로드는 되지만 비용이 늘고 정작 필요한 줄이 묻힙니다. 넘치는 내용은 지우거나, 일부 경로에만 해당하면 조건부 룰로(5.3절), 절차라면 Skill로(5.4절) 옮깁니다. 한 줄을 더하는 시점은 2.4절의 제1원칙이 정합니다. 에이전트가 같은 실수를 두 번째로 했을 때입니다. 다만 지침은 어디까지나 요청입니다. Claude Code 문서도 지침을 "context, not enforced configuration"(강제 설정이 아니라 맥락)이라고 적습니다. 반드시 지켜져야 하는 것은 5.5절의 Hook으로 내립니다.

#### 이미 써 둔 지침을 줄이는 절차

2.5절의 변화는 새로 쓸 때만의 이야기가 아닙니다. **이미 써 둔 지침과 스킬에도 그대로 해당합니다.** 구세대 모델에 맞춰 쌓아 둔 규칙은 지금 모델에서 과잉 제약이 되고, 그 사이에 경로나 명령이 바뀌어 낡아 버린 줄도 섞입니다. 두 회사의 권고는 "처음부터 새로 쓰라"가 아닙니다. 지우고 재는 것을 **한 묶음씩** 반복하라는 것입니다.

1. **감사합니다.** Claude Code에는 지침 파일을 검사하는 명령이 있습니다. `/doctor prompt-audit`은 **구세대 모델에 맞춰 쓴 지시, 존재하지 않는 파일이나 명령에 대한 참조, 서로 모순되는 파일**을 찾아 제안과 함께 보고하고, 사용자가 적용을 요청할 때까지 파일을 바꾸지 않습니다. 기본 검사 범위는 `CLAUDE.md`, `CLAUDE.local.md`, `AGENTS.md`와 `.claude/` 아래의 룰·스킬·명령·서브에이전트입니다. 경로를 주면 한 곳만 검사합니다(`/doctor prompt-audit .claude/skills/deploy`). v2.1.283 이상이 필요합니다(출처: [Claude Code 문서, Audit your instruction files](https://code.claude.com/docs/en/memory#audit-your-instruction-files), 2026-10 확인).
2. **한 묶음씩 지웁니다.** OpenAI 문서의 권고는 "Remove one group of instructions, examples, or tools at a time, then rerun the same evals."(지시나 예시, 도구를 한 묶음씩 지우고 같은 평가를 다시 돌려라)이고, 프롬프트 더미 전체를 한 번에 다시 쓰지는 말라고 합니다. 한꺼번에 바꾸면 무엇이 효과였는지 알 수 없습니다.
3. **같은 과제로 다시 잽니다.** 대표 과제 몇 개를 정해 두고 지우기 전후를 비교합니다. 측정 방법은 부록 C.4에 있습니다. 이 단계가 없으면 지침 다이어트는 취향이 됩니다.

무엇을 남길지는 미리 정해 둡니다. 아래 표는 두 회사의 권고를 합친 것입니다.

| 지운다 | 남긴다 |
| --- | --- |
| 모델이 코드와 파일 시스템을 보면 알 수 있는 것 (디렉터리 구조, 의존성 목록) | 사용자에게 보이는 결과, 완료 기준과 정지 조건 |
| 같은 금지를 여러 곳에서 되풀이한 줄 | 도메인 제약과 안전 규칙 |
| 행동을 바꾸지 않는 예시와 문체 지시 | 상황에 따라 달라지는 도구 선택 규칙 |
| 서로 부딪치는 지시 | 요구되는 출력 형식과 검증 조건 |
| 낡은 경로, 없어진 명령, 옛 모델 이름 | 코드를 읽어도 드러나지 않는 프로젝트 고유의 함정 |

(출처: [OpenAI, Prompting guidance for GPT-5.6 Sol](https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6)의 "Simplify prompts first"와 [Anthropic, The new rules of context engineering for Claude 5 generation models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models), 2026-10 확인)

어법도 함께 손봅니다. OpenAI 문서는 `ALWAYS`, `NEVER`, `must`를 안전 규칙이나 필수 항목처럼 **예외가 없는 불변**에만 쓰고, 판단이 필요한 것에는 판단 기준을 적으라고 합니다. 위의 "주변 코드처럼 읽히게 쓴다"가 그 형태입니다. 그리고 절대 어법을 써야 할 만큼 중요한 줄이라면, 그것은 애초에 글이 아니라 Hook으로 내려야 하는 줄일 수 있습니다(5.5절).

Codex CLI 쪽에도 같은 성격의 절차가 있습니다. OpenAI의 최신 모델 안내서는 스킬과 `AGENTS.md` 같은 파일을 감사하라고 권하고, Codex에서 `$openai-docs migrate this project to the GPT-6 model family`로 권고 변경을 적용할 수 있다고 적습니다(출처: [OpenAI, Latest model prompting best practices](https://developers.openai.com/api/docs/guides/latest-model), 2026-10 확인).

> **발표 전 확인**: 사내 환경에서 `/doctor prompt-audit`이 동작하는지 확인합니다. 이 감사는 번들 스킬을 거치므로, 번들 스킬을 끈 설정에서는 쓸 수 없습니다. 실습용 리포지토리의 지침에 일부러 낡은 줄과 모순되는 줄을 넣어 두고 보고서를 화면으로 보여 주면 이 절의 설명이 한 번에 전달됩니다.

### 5.3 필요할 때만 읽히게 하기

지침이 커지면 모든 세션의 모든 호출에 비용이 붙습니다. 그래서 항상 필요하지는 않은 내용은 **필요할 때만 읽히게** 옮깁니다. 지연 로드가 되는 방법은 세 가지입니다.

1. **`paths` 조건부 룰.** `.claude/rules/` 아래 파일의 머리말(frontmatter)에 `paths:`로 `src/can/**/*.c` 같은 패턴을 적으면, Claude가 그 패턴에 맞는 파일을 읽거나 고칠 때만 로드됩니다.
2. **하위 디렉터리 지침.** 작업 디렉터리 아래의 `CLAUDE.md`는 시작할 때 로드되지 않고, Claude가 그 디렉터리의 파일을 읽거나 고칠 때 로드됩니다.
3. **경로만 적고 "필요하면 읽어라"고 안내하기.** 지침에 경로를 일반 텍스트나 링크로 적습니다. 경로 한 줄만 상주하고, 내용은 Claude가 그 파일을 실제로 읽을 때 들어옵니다.

#### `@path`는 지연 로드가 아니다

흔한 오해가 하나 있습니다. "지침에 `@경로`를 적으면 경로만 올라가고 내용은 나중에 읽힌다"는 것입니다. 사실은 반대입니다. `@`를 붙이면 그 파일의 내용이 세션을 시작할 때 펼쳐져 함께 로드됩니다. 적는 방식에 따라 내용이 올라가는 시점이 다릅니다.

| 적는 방식 | 지침 파일에서의 의미 | 내용이 컨텍스트에 올라가는 시점 |
| --- | --- | --- |
| `@docs/coding-standard.md` | import | 세션 시작 때 내용 전체 |
| `docs/coding-standard.md` 또는 링크, "필요하면 읽어라" | 단순 언급 | Claude가 그 파일을 읽을 때 |
| `.claude/rules/*.md`에 `paths:` 지정 | 조건부 룰 | 해당 파일을 다룰 때 |
| 하위 디렉터리의 `CLAUDE.md` | 중첩 지침 | 그 경로의 파일을 읽을 때 |

공식 문서의 문장은 다음과 같습니다.

> "Imported files are expanded and loaded into context at launch alongside the CLAUDE.md that references them."
> (import한 파일은 그것을 참조한 `CLAUDE.md`와 함께 시작 시점에 펼쳐져 컨텍스트에 로드된다.)

> "Imports help you organize a long file but don't reduce its context cost, because imported files also load at launch."
> (import는 긴 파일을 정리하는 데는 도움이 되지만 컨텍스트 비용을 줄이지는 않는다. import한 파일도 시작할 때 로드되기 때문이다.)

> "To mention a path in your CLAUDE.md without importing it, wrap it in backticks: writing `` `@README` `` keeps the text literal, while `@README` outside backticks imports the file."
> (`CLAUDE.md`에서 경로를 import하지 않고 언급만 하려면 백틱으로 감싼다. `` `@README` ``라고 쓰면 글자 그대로 남고, 백틱 밖의 `@README`는 그 파일을 import한다.)

정리하면 `@path`는 긴 지침을 여러 파일로 나누는 **정리용**이지 절감용이 아닙니다. 지연 로드를 원하면 `@`를 떼고 경로만 적거나 백틱으로 감쌉니다. 5.2절 예시에서 `docs/misra-deviations.md`를 백틱으로 적은 이유가 이것입니다.

스킬도 같은 원리로 컨텍스트를 줄입니다. 공식 문서의 스킬 예시는 보조 파일을 `@`가 아니라 `[reference.md](reference.md)` 같은 일반 링크로 가리키고, 그 파일을 "loaded when needed"(필요할 때 로드된다)라고 설명합니다(5.4절). 스킬 본문에 적은 `@` 참조는 스킬이 실행될 때 그 파일을 함께 첨부하는 것으로 읽힙니다. 다만 이 동작을 직접 정의한 절은 스킬 문서에 없고 근거는 간접적입니다. claude.ai에서 동기화한 스킬은 로컬 스킬과 달리 `@` 참조가 가리키는 파일을 첨부하지 않는다는 문장과, 기능 비교표에 스킬도 `@path` import로 파일을 포함할 수 있다고 적힌 칸이 전부입니다.

**확인 방법.** 세션을 시작할 때 무엇이 로드됐는지는 `/context`의 `Memory files` 목록에서 봅니다. 공식 문서가 지목하는 수단이 이것입니다("To check which `CLAUDE.md` and rules files loaded at launch, run `/context`."). `/memory`는 지침 파일의 위치를 나열하고 편집기로 여는 명령입니다. 하위 디렉터리 지침은 `Memory files`에 나오지 않고, 로드되는 순간 터미널에 `Loaded` 줄이 나옵니다.

> **발표 전 확인**: 지침에 `@docs/…` 한 줄을 넣기 전과 후에 새 세션에서 `/context`를 실행해 `Memory files`가 어떻게 달라지는지 화면을 준비합니다. import한 파일이 목록에 별도 항목으로 나오는지는 문서에 적혀 있지 않습니다. 문서로 뒷받침되는 관측 수단은 `InstructionsLoaded` 훅이며, import로 로드된 파일은 `load_reason`이 `include`로 기록됩니다.

출처는 다음과 같습니다.

- Claude Code 문서, `CLAUDE.md`의 import: <https://code.claude.com/docs/en/memory#import-additional-files>
- Claude Code 문서, 스킬의 보조 파일: <https://code.claude.com/docs/en/skills#add-supporting-files>
- 위 두 문서의 미러(사내망에서 공식 사이트에 접근할 수 없을 때): <https://github.com/posimind/claude-code-docs/blob/main/docs/memory.md>, <https://github.com/posimind/claude-code-docs/blob/main/docs/skills.md>

#### 지연 로드의 대가

지연 로드는 공짜가 아닙니다. 대가가 두 가지 있습니다. 그래서 어느 작업에서나 지켜야 하는 것은 루트 지침에 짧게 두고, 특정 경로나 특정 작업에서만 필요한 것을 지연 로드로 내립니다.

- **압축 때 사라집니다.** `paths` 룰과 하위 디렉터리 지침은 대화 기록 쪽에 실립니다. 그래서 압축이 일어나면 다른 대화와 함께 요약돼 빠지고, 해당 파일을 다시 다룰 때에야 돌아옵니다(1.3절). 프로젝트 루트의 지침은 압축 뒤에 디스크에서 다시 주입됩니다. 문서의 권고는 "If a rule must persist across compaction, drop the `paths:` frontmatter or move it to the project-root CLAUDE.md."(압축을 넘어 유지돼야 하는 규칙이면 `paths:`를 빼거나 루트의 `CLAUDE.md`로 옮긴다)입니다(출처: [Claude Code 문서, Explore the context window](https://code.claude.com/docs/en/context-window#what-survives-compaction), 2026-10 확인).
- **셋째 방법은 Claude가 읽기로 정해야 읽힙니다.** 경로만 적어 둔 파일은 Claude가 열지 않으면 없는 것과 같습니다. 그러니 언제 읽어야 하는지를 경로 옆에 함께 적습니다. "ISR이나 드라이버를 고칠 때는 `docs/isr-rules.md`를 먼저 읽는다"처럼 씁니다.

#### 여러 도구를 쓸 때의 단일 원본

도구를 두 가지 이상 쓰면 지침 파일이 도구 수만큼 생기고, 내용이 갈라지기 시작하면 관리할 수 없습니다. 원본은 **`AGENTS.md` 하나**로 두고 나머지가 그것을 읽게 합니다(3.5절). Copilot CLI는 `AGENTS.md`와 `CLAUDE.md`를 모두 읽습니다. Codex CLI와 opencode는 `AGENTS.md`를 기본으로 읽고, Codex CLI는 `CLAUDE.md`를 기본으로는 읽지 않으며 opencode도 V2부터 읽지 않습니다(출처: 각 도구의 공식 문서, 2026-10 확인. 도구별 비교는 3.2절).

Claude Code에만 줄 지침이 따로 있으면 `CLAUDE.md`를 만들어 첫 줄에 `@AGENTS.md`를 적고 그 아래에 Claude Code 전용 내용을 씁니다. 여기서는 `@`가 맞는 용도입니다. 어차피 시작할 때 전부 읽혀야 하는 내용이기 때문입니다. 예전에 쓰던 `ln -s AGENTS.md CLAUDE.md` 같은 심볼릭 링크는 이제 `AGENTS.md`를 직접 읽지 못하는 이전 버전이나 일부 세션에서만 필요합니다.

지연 로드 기능은 도구마다 형식이 다르다는 점도 봐 둡니다. 경로 조건부 지침은 Claude Code(`paths:`)와 Copilot(`applyTo`)에 있고 형식이 서로 다르며, Codex CLI와 opencode에는 없습니다. `@`를 처리하는 방식도 제각각입니다. 그래서 원본 `AGENTS.md`에는 어느 도구에서나 뜻이 같은 일반 마크다운만 씁니다. 경로를 적고 언제 읽을지 안내하는 셋째 방법은 도구를 옮겨도 그대로 통합니다.

### 5.4 Skill

Skill은 지식이나 절차를 담은 `SKILL.md` 파일입니다. 지침 파일과 다른 점은 로드 방식입니다. **평소에는 설명 한 줄만 컨텍스트에 있고, 쓸 때만 본문이 들어옵니다.** 500줄짜리 내용을 지침에 두면 매 호출에 500줄이 실리지만, Skill에 두면 평소에는 한 줄입니다. 사람이 `/isr-review`처럼 직접 부르거나, 모델이 설명을 보고 관련이 있다고 판단하면 스스로 불러옵니다.

최소 구조는 다음과 같습니다. 위쪽 머리말(frontmatter)의 `description`이 평소에 실리는 한 줄이고, 그 아래가 쓸 때 들어오는 본문입니다.

```markdown
---
name: isr-review
description: ISR이나 드라이버 코드(.c, .h)를 리뷰하거나 고치기 전에 쓴다. 블로킹 호출, volatile 누락, 공유 변수 보호, MISRA 위반을 점검한다.
---

## 절차
1. `git diff --name-only`로 바뀐 파일 목록을 본다
2. 파일마다 아래 체크리스트를 적용하고, 위반을 `파일:줄` 형식으로 보고한다. 고치지는 않는다

## 체크리스트
- ISR 안에 블로킹 호출이 없는가
- ISR과 태스크가 함께 쓰는 변수가 임계 구역으로 보호되는가

## 참고
- 승인된 MISRA 예외는 [misra-deviations.md](misra-deviations.md)에 있다. 규칙 위반으로 보이는 코드를 만나면 읽는다
```

잘 쓰는 요령은 세 가지입니다.

- **`description`이 라우팅 키입니다.** 모델은 이 한 줄만 보고 스킬을 쓸지 정합니다. 그래서 "무엇인지"보다 "언제 써야 하는지"를 적습니다. 설명이 모호하거나 다른 스킬과 겹치면 엉뚱한 스킬이 로드되거나 필요한 스킬이 걸리지 않습니다. 목록에 실리는 글은 스킬마다 `description`과 `when_to_use`를 합쳐 1,536자에서 잘리므로 핵심 용례를 앞에 둡니다.
- **보조 파일은 `SKILL.md`에서 링크로 가리킵니다.** 위 예시의 마지막 줄이 그것입니다. 긴 참고 자료는 별도 파일에 두고 링크와 함께 언제 읽을지를 적으면 필요할 때만 읽힙니다. 5.3절의 셋째 방법과 같은 원리입니다. 문서는 `SKILL.md`를 500줄 아래로 유지하라고 권합니다. 한번 로드된 본문은 그 뒤 호출에 계속 실리기 때문입니다.
- **스킬도 다이어트 대상입니다.** 과잉 제약은 지침 파일보다 스킬에서 더 눈에 띄지 않습니다. 평소에는 설명 한 줄만 보이기 때문입니다. `/doctor prompt-audit`의 기본 검사 범위에 스킬이 들어 있고(5.2절), OpenAI 문서는 스킬 파일의 불명확하거나 충돌하는 안내가 모델을 작업 중에 일찍 멈추게 할 수 있다고 적습니다.
- **부작용이 있는 스킬은 사람만 부르게 합니다.** 배포나 전송처럼 실행 시점을 사람이 정해야 하는 스킬에는 `disable-model-invocation: true`를 넣습니다. 이 스킬은 설명도 컨텍스트에 실리지 않고, 사람이 `/이름`으로 부를 때만 동작합니다.

(출처: [Claude Code 문서, Skills](https://code.claude.com/docs/en/skills), 2026-10 확인)

Skill은 지침과 마찬가지로 **요청일 뿐 강제가 아닙니다.** 모델이 읽고 해석하므로 결과가 흔들리고, 세션이 길어지면 처음에는 따르던 스킬을 나중에는 놓치기도 합니다. 문서의 처방은 분명합니다. 매번 지켜져야 하는 규칙을 Claude가 건너뛰었다면 그 규칙을 Hook으로 옮깁니다. Skill은 Agent Skills라는 공개 규격을 따르므로, 표준 필드와 일반 마크다운만 쓴 스킬은 다른 도구로 옮겨집니다(3.5절).

### 5.5 Hook

Hook은 하네스가 정해진 시점에 자동으로 실행하는 명령입니다. 도구를 실행하기 직전, 파일을 고친 직후, 세션을 시작할 때 같은 이벤트에 스크립트를 걸어 둡니다. 실행하는 쪽이 모델이 아니라 하네스이므로 모델이 잊거나 건너뛸 수 없고, 출력을 돌려주지 않으면 컨텍스트 비용도 없습니다. Claude Code 문서는 지침과의 차이를 이렇게 적습니다.

> "An instruction like "never edit `.env`" in CLAUDE.md or a skill is a request, not a guarantee. A `PreToolUse` hook that blocks the edit is enforcement."
> (`CLAUDE.md`나 스킬에 적은 "`.env`를 절대 고치지 마라"는 요청이지 보장이 아니다. 그 편집을 막는 `PreToolUse` 훅이 강제다.)
> 출처: [Claude Code 문서, Extend Claude Code](https://code.claude.com/docs/en/features-overview#compare-similar-features), 2026-10 확인

쓰는 자리는 두 곳이 대표적입니다. **편집 후 자동 검증**(`PostToolUse`)과 **위험 동작 차단**(`PreToolUse`)입니다. 예시를 보기 전에 규격 세 가지를 짚습니다.

- **입력은 표준 입력(stdin)으로 오는 JSON입니다.** 고친 파일 경로는 `tool_input.file_path`에, 실행하려는 셸 명령은 `tool_input.command`에 들어 있습니다. 파일 경로를 담은 환경 변수는 없습니다.
- **종료 코드 2가 차단입니다.** 0은 문제없음이고, 2는 차단이며 stderr에 쓴 글이 모델에게 사유로 전달됩니다. 1을 포함한 나머지 값은 차단이 아닙니다. 오류 알림만 남기고 동작은 그대로 진행됩니다.
- **`PreToolUse`에서 막는 방법은 두 가지입니다.** 종료 코드 2로 끝내거나, 종료 코드 0으로 끝내면서 `hookSpecificOutput.permissionDecision`을 담은 JSON을 출력합니다. 값은 `deny`(거부) 외에 `ask`(사람에게 묻기)도 있습니다.

#### 예시 ① 편집 후 자동 검증

`.c`나 `.h` 파일이 편집될 때마다 정적 검사를 돌립니다. 설정은 프로젝트의 `.claude/settings.json`에 둡니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-edited-file.sh", "args": [] }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/check-edited-file.sh  (PostToolUse, matcher: Edit|Write)
INPUT=$(cat)                                   # 입력은 stdin으로 오는 JSON이다
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [[ "$FILE_PATH" != *.c && "$FILE_PATH" != *.h ]]; then
  exit 0                                       # C 소스와 헤더만 검사한다
fi

# 검사 명령은 프로젝트의 것으로 바꾼다 (포매터, MISRA 체커 등)
if ! RESULT=$(gcc -fsyntax-only -Wall -Werror "$FILE_PATH" 2>&1); then
  echo "정적 검사 실패: $FILE_PATH" >&2
  echo "$RESULT" >&2
  exit 2                                       # 편집은 이미 끝났다. stderr가 모델에게 전달된다
fi
exit 0
```

검사가 실패하면 컴파일러의 오류 메시지가 모델에게 돌아가고, 모델은 다음 회전에서 그것을 고칩니다. 2.3절의 계산적 센서가 루프 안으로 들어온 것입니다. `gcc` 옵션은 자리를 채운 예일 뿐이고, 실제로는 팀의 포매터와 정적 분석기, MISRA 체커를 부릅니다. `PostToolUse`는 이미 끝난 편집을 되돌리지 못합니다. 모델이 셸 명령으로 파일을 바꾸면 이 훅은 돌지 않습니다.

#### 예시 ② 위험 동작 차단

같은 `hooks` 객체에 `PreToolUse` 키를 더하고 `matcher`를 `Bash`로 둔 뒤 아래 스크립트를 겁니다. 모든 셸 명령이 실행되기 전에 이 스크립트를 거칩니다.

```bash
#!/bin/bash
# .claude/hooks/block-dangerous.sh  (PreToolUse, matcher: Bash)
COMMAND=$(jq -r '.tool_input.command // empty')

deny() {
  jq -n --arg reason "$1" '{hookSpecificOutput: {hookEventName: "PreToolUse",
    permissionDecision: "deny", permissionDecisionReason: $reason}}'
  exit 0
}

if echo "$COMMAND" | grep -qE 'rm -(rf|fr)'; then
  deny "rm -rf는 프로젝트 훅이 막는다. 지울 파일을 하나씩 지정한다."
fi
if echo "$COMMAND" | grep -qE 'git push.* (--force|-f)( |$)'; then
  deny "강제 push는 프로젝트 훅이 막는다. 필요하면 사람에게 요청한다."
fi
exit 0                                         # 결정 없음. 일반 권한 흐름으로 넘어간다
```

거부 사유는 모델에게 전달됩니다. 사유에 다음 행동을 적어 두면 모델이 다른 길을 찾습니다. 같은 자리에서 타깃 보드에 굽는 명령이나 사외로 나가는 전송도 막을 수 있습니다. `deny` 대신 `ask`를 돌려주면 사람에게 승인 프롬프트가 뜹니다. 테스트 파일이나 검사 스크립트의 수정처럼, 막을 수는 없지만 사람이 봐야 하는 변경에 씁니다(13.4절).

> **발표 전 확인**: 두 스크립트는 표본 JSON을 stdin으로 넣어 종료 코드와 출력만 시험했습니다. 실제 세션에 등록해 편집과 `rm -rf`를 시켜 보고, 훅이 발동하는지와 모델이 사유를 받는지 확인합니다.

#### 한계와 주의

- **문자열 검사는 보안 경계가 아닙니다.** 위 스크립트는 `rm -r -f`나 `git -C . push --force` 같은 다른 표기를 놓칩니다. 모델이 평소에 쓰는 표기를 막는 데는 쓸 만하지만 의도적인 우회는 막지 못합니다. 명령 글자에 기대지 않는 강제는 샌드박스와 권한 설정으로 합니다(7.5절).
- **훅이 실패하면 통과됩니다.** 스크립트 경로를 잘못 적거나 스크립트가 죽으면 오류 알림만 남고 동작은 진행됩니다. 정책용 훅은 등록한 뒤 실제로 걸리는지 한 번 확인합니다. 실패했을 때도 막으려면 훅에 `"onFailure": "block"`을 겁니다(v2.1.295 이상).
- **훅은 내 권한으로 실행되는 코드입니다.** 저장소에 커밋된 훅은 코드 리뷰 대상입니다. 훅이 돌려준 글은 컨텍스트에 쌓이므로 긴 로그 대신 실패한 항목과 고치는 법만 돌려줍니다.
- **설정 위치와 지원 범위는 도구마다 다릅니다.** Copilot CLI는 `.claude/settings.json`의 훅을 읽고 Codex CLI는 구조가 비슷한 자기 설정 파일을 쓰지만, opencode에는 설정 파일로 거는 훅이 없습니다(3.2절, 3.5절). 검사 로직은 스크립트 파일에 두고 훅 설정은 그것을 부르는 얇은 껍데기로 두면, 도구를 옮길 때 고칠 것이 적습니다.

(출처: [Claude Code 문서, Hooks reference](https://code.claude.com/docs/en/hooks), [Hooks guide](https://code.claude.com/docs/en/hooks-guide), [Permissions](https://code.claude.com/docs/en/permissions#bash-rule-limits), 2026-10 확인)

이제 이 장의 질문에 답할 수 있습니다. 2.4절의 "ISR 안에서 `printf`를 쓰지 않는다"를 세 부품에 놓아 봅니다.

| 규칙의 성격 | 둘 곳 | `printf` 금지의 경우 |
| --- | --- | --- |
| 어느 작업에서나 알아야 하는 사실과 판단 기준 | 지침 파일 | "ISR 안에서는 블로킹 호출을 쓰지 않는다" 한 줄 |
| 특정 작업이나 특정 경로에서만 필요한 절차와 체크리스트 | Skill, 조건부 룰 | ISR 리뷰 스킬의 체크리스트 항목 |
| 매번 반드시 일어나야 하거나 절대 일어나면 안 되는 것 | Hook | 편집 직후 ISR 파일에서 `printf`를 찾는 검사 |

셋은 하나를 고르는 관계가 아닙니다. 지침으로 알려 주고 Hook으로 확인합니다. 처음부터 전부 만들 필요도 없습니다. 같은 실수가 다시 나오면 한 칸씩 아래로 내립니다.

> **질문**: 지금 팀의 지침 파일에서 "반드시"나 "절대"가 붙은 줄을 하나 고릅니다. 그 줄은 지금 요청인가요, 강제인가요?

### 핵심 정리

- 지침 파일에는 모델이 추측할 수 없는 명령, 판단 기준, 프로젝트 고유의 함정만 짧게 적고, 나머지는 조건부 룰과 Skill로 내려 필요할 때만 읽히게 합니다.
- `@path`는 세션을 시작할 때 내용까지 로드하는 정리용이므로, 지연 로드를 원하면 `@` 없이 경로만 적고 언제 읽을지를 함께 적습니다.
- 지침과 Skill은 요청이고 Hook은 강제이므로, 반드시 지켜져야 하는 규칙은 Hook으로 내립니다.

**다음 장으로**: 프로젝트 밖의 시스템에 닿거나, 큰 조사를 맡기려면 무엇이 필요한가요

---

## 6장. 확장 부품 ②: MCP · Subagent · Plugin

- **핵심 질문**: 외부 시스템 연결과 작업 위임에는 어떤 위험과 대가가 따르나요
- **도달 목표**: MCP를 붙일 때 보안을 점검하고, 서브에이전트에 맡길 일을 구분합니다

5장의 세 부품은 프로젝트 안에서 일했습니다. 이 장의 세 부품은 경계를 넘습니다. MCP는 프로젝트 밖의 시스템에 닿게 하고, Subagent는 큰 조사를 다른 컨텍스트에 맡기게 하며, Plugin은 만든 부품을 다른 저장소와 다른 사람에게 나눠 줍니다. 경계를 넘는 만큼 위험과 대가가 따릅니다.

MCP의 컨텍스트 비용과 tool search는 4장에서 이미 다뤘습니다. 여기서는 반복하지 않고 4.4\~4.8절을 참조합니다. 이 장이 MCP에 대해 다루는 것은 개념과 보안입니다.

### 6.1 MCP란

MCP(Model Context Protocol)는 AI 애플리케이션과 외부 도구·데이터를 잇는 공개 프로토콜입니다. 에이전트마다, 시스템마다 따로 만들던 N×M개의 연결을 N+M개로 줄이는 것이 목적입니다. 구조는 세 역할로 이뤄집니다.

```text
Host (Claude Code)
 ├─ Client ── MCP Server A (이슈 트래커)
 ├─ Client ── MCP Server B (사내 문서 검색)
 └─ Client ── MCP Server C (빌드 서버)
```

| 역할 | 무엇인가 | 우리 환경에서 |
| --- | --- | --- |
| Host | 클라이언트들을 조율하는 AI 애플리케이션. 컨텍스트에 무엇을 넣고 무엇을 허용할지 정합니다 | Claude Code, Codex CLI 같은 코딩 에이전트 |
| Client | 호스트 안에서 서버 하나와의 연결을 맡는 부분. 서버마다 하나씩 생깁니다 | 도구에 내장돼 있습니다 |
| Server | 컨텍스트와 기능을 제공하는 프로그램. 내 PC의 프로세스(STDIO)이거나 원격 서버(HTTP)입니다 | 이슈 트래커, 문서 검색, 빌드 서버 연결 |

서버가 내놓는 것은 세 종류이고, 종류마다 꺼내 쓰는 주체가 다릅니다.

| 서버가 내놓는 것 | 무엇인가 | 누가 꺼내 쓰는가 |
| --- | --- | --- |
| Tools | 실행할 수 있는 함수. API 호출, 파일 쓰기 | **모델**이 스스로 판단해 호출합니다 |
| Resources | 맥락으로 쓸 데이터. 파일 내용, 레코드 | **애플리케이션**(호스트)이 붙입니다 |
| Prompts | 재사용하는 프롬프트 템플릿 | **사용자**가 고릅니다. 보통 슬래시 명령입니다 |

(출처: [MCP 문서, Architecture overview](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture), [MCP 스펙 2026-07-28, Server 개요](https://modelcontextprotocol.io/specification/2026-07-28/server), 2026-10 확인)

셋 중 가장 많이 쓰이고 가장 조심할 것이 Tools입니다. 모델이 판단해서 부르기 때문입니다. 스펙도 Tools 항목에 "there **SHOULD** always be a human in the loop with the ability to deny tool invocations"(도구 호출을 거부할 수 있는 사람이 언제나 루프 안에 있어야 한다)라고 적습니다. 이 문장을 구현하는 자리가 7.5절의 권한 모드입니다.

MCP로 얻는 것은 두 가지입니다.

- **표준화.** 한 번 만든 서버를 Claude Code, Codex CLI, Copilot, opencode가 함께 씁니다. 도구를 갈아타도 서버는 그대로 남습니다(3.5절).
- **자격증명 격리.** 토큰은 서버가 쥐고, 모델이 보는 것은 도구 정의와 호출 결과뿐입니다. 서버 코드에 검사를 넣으면 모델이 우회할 수 없는 가드레일도 됩니다(7.2절). 다만 격리의 정도는 서버를 어디서 돌리느냐에 달려 있습니다. 내 PC에서 도는 STDIO 서버는 에이전트와 같은 사용자 권한으로 실행됩니다.

반대로 **쓰지 말아야 할 때**도 분명합니다.

- **CLI로 이미 되는 일.** `git`, `gh`, `curl`로 되는 일에는 서버가 필요 없습니다. Claude Code 문서도 CLI 도구가 MCP 서버보다 컨텍스트를 덜 쓴다고 적습니다. 도구 목록을 따로 싣지 않기 때문입니다(출처: [Claude Code 문서, Manage costs effectively](https://code.claude.com/docs/en/costs#reduce-mcp-server-overhead), 2026-10 확인).
- **가끔 쓰는데 도구 정의가 무거운 서버.** tool search가 꺼진 환경에서는 쓰지 않아도 매 호출에 실립니다(4.4절).
- **보안 검토가 끝나지 않은 남의 서버.** 이유는 다음 절에서 봅니다.

붙일 만한 신호는 하나입니다. 다른 시스템의 화면에서 데이터를 복사해 채팅에 붙여 넣는 일이 반복될 때입니다.

### 6.2 MCP 보안

MCP 서버를 붙인다는 것은 두 가지를 허락한다는 뜻입니다. 남이 만든 코드를 내 PC에서 내 권한으로 실행하고, 그 코드가 가져온 글을 모델의 컨텍스트에 넣습니다. 위협도 이 두 갈래에서 나옵니다.

| 위협 | 무슨 일이 일어나는가 | 확인된 사례와 근거 |
| --- | --- | --- |
| 공급망 | 설치한 서버 패키지 자체가 악성이거나, 정상이던 패키지가 나중에 바뀝니다 | 가짜 `postmark-mcp` 패키지(2025-09), Oura MCP 서버 복제본(2026-02) |
| STDIO 명령 주입 | 서버 설정에 적힌 명령이 그대로 내 PC에서 실행됩니다. 설정을 바꿀 수 있으면 임의의 명령을 실행할 수 있습니다 | OX Security의 공개(2026-04-15) |
| 간접 프롬프트 인젝션 | 도구가 읽어 온 이슈, 문서, 웹페이지에 심긴 지시를 모델이 따릅니다. 도구 설명에 지시를 숨기는 변형도 있습니다 | 원리상 모든 도구 결과에 해당합니다 |

**공급망.** 실제 사고가 있었습니다. 이메일 서비스 Postmark의 공지에 따르면, 공식 도구를 사칭한 npm 패키지 `postmark-mcp`가 15개 버전 동안 정상으로 동작하다가 1.0.16 버전에서 모든 메일을 외부로 숨은 참조(BCC)하는 한 줄을 넣었습니다(출처: [Postmark 공지](https://postmarkapp.com/blog/information-regarding-malicious-postmark-mcp-package), 2025-09-25). 보안 업체 Straiker는 정상 Oura MCP 서버를 복제해 가짜 포크와 기여자로 신뢰를 꾸민 뒤 정보 탈취 악성코드 StealC를 심은 사례를 보고했습니다(출처: [Straiker STAR Labs](https://www.straiker.ai/blog/smartloader-clones-oura-ring-mcp-to-deploy-supply-chain-attack), 2026-02-05. 발견한 업체 한 곳의 보고입니다). 처음 설치할 때 본 코드가 다음 업데이트에도 같다는 보장이 없다는 것이 두 사례의 교훈입니다.

**STDIO 명령 주입.** 로컬 MCP 서버는 설정 파일에 적힌 명령을 실행해서 띄웁니다. 2026년 4월 OX Security는 이 구조에서 나오는 명령 실행 문제를 공개했습니다. 인용할 때는 출처를 구분합니다.

- 규모 수치(다운로드 1억 5천만 회 이상, 공개 서버 7,000개 이상, 취약 인스턴스 최대 20만 개)는 OX의 메인 글에 적힌 **OX 자체 추정**입니다. 독립 검증은 없고, 20만은 공개 IP에서 찾은 서버 수로부터 외삽한 값이라고 보도됐습니다(출처: [OX Security](https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-critical-systemic-vulnerability-at-the-core-of-the-mcp/), 2026-04-15. 외삽이라는 설명은 [VentureBeat](https://venturebeat.com/security/mcp-stdio-flaw-200000-ai-agent-servers-exposed-ox-security-audit), 2026-05-01 보도).
- 같은 날 나온 보안 권고 페이지에는 수치가 없습니다. 대신 우회 사례가 있습니다. 실행 파일을 `python`, `npm`, `npx`로 제한해도 `npx -c <명령>`처럼 인자로 명령을 넣을 수 있었습니다(출처: [OX Security Advisory](https://www.ox.security/blog/mcp-supply-chain-advisory-rce-vulnerabilities-across-the-ai-ecosystem/), 2026-04-15).
- OX는 Anthropic이 이 동작을 예상된 것이라 답하고 프로토콜 구조 변경을 거절했다고 설명합니다. Anthropic의 직접 성명은 확인되지 않습니다. 다만 MCP 스펙 저장소의 `SECURITY.md`가 STDIO 전송의 명령 실행을 "an intended feature, not a vulnerability"(취약점이 아니라 의도된 기능)라고 적고 있습니다(출처: [MCP SECURITY.md](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/SECURITY.md), 2026-10 확인).

우리에게 남는 결론은 책임의 위치입니다. 설정에 적힌 명령을 실행하는 것이 프로토콜의 의도라면, 그 설정을 누가 바꿀 수 있는지를 지키는 일은 쓰는 쪽의 몫입니다. OX의 권고에 따르면 Claude Code와 GitHub Copilot은 MCP 설정 파일을 고치기 전에 사용자 승인을 한 번 요구합니다. 그 승인 창을 읽지 않고 누르면 막는 것이 없습니다.

**간접 프롬프트 인젝션.** 모델은 컨텍스트에 들어온 글이 어디서 왔는지를 믿을 만하게 구분하지 못합니다. Simon Willison의 표현으로는 "LLMs are unable to reliably distinguish the importance of instructions based on where they came from."(LLM은 지시가 어디서 왔는지에 따라 그 중요도를 믿을 만하게 구분하지 못한다)입니다. 그는 세 능력이 한 에이전트에 모이면 데이터 탈취의 조건이 갖춰진다고 정리했습니다. 비공개 데이터 접근, 신뢰할 수 없는 콘텐츠에의 노출, 외부로 내보내는 통신입니다(출처: [Willison, The lethal trifecta for AI agents](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/), 2025-06-16). MCP 서버를 하나 더 붙일 때마다 이 셋 중 무엇이 더해지는지를 봅니다.

그래서 이 세미나는 원칙을 한 문장으로 둡니다. **도구가 읽어 온 것은 데이터이지 명령이 아닙니다.** 인용문이 아니라 세미나가 정한 원칙 문장입니다. 근거는 위 Willison의 문장과, [MCP 문서 Client Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/develop/clients/client-best-practices)의 "Tool results from one server are untrusted input to another."(한 서버의 도구 결과는 다른 서버에 신뢰할 수 없는 입력이다)입니다.

```text
사용자가 채팅으로 준 지시           →  명령
파일·웹페이지·이슈에서 읽어 온 것   →  데이터
```

모델이 이 구분을 항상 지킨다는 보장은 없습니다. 그래서 원칙을 모델에게 맡겨 두지 않고 설정과 승인으로 받칩니다. MCP 서버를 붙이기 전에 아래를 점검합니다.

| 점검 | 무엇을 보는가 |
| --- | --- |
| 출처 | 직접 만들었거나 신뢰하는 제공자의 서버인가요? Claude Code 문서는 Anthropic이 어떤 MCP 서버도 보안 감사하거나 관리하지 않는다고 적습니다 |
| 버전 | `latest`가 아니라 특정 버전으로 고정했나요? 업데이트할 때 바뀐 내용을 보나요 |
| 실행 명령 | 설정 파일의 명령과 인자를 끝까지 읽었나요? 설정을 바꾸려는 승인 창을 읽고 누르나요 |
| 권한 | 서버에 준 토큰의 범위가 필요한 만큼인가요? 읽기만 하면 되는 서버에 쓰기 권한을 주지 않았나요 |
| 세 능력의 조합 | 비공개 데이터 접근, 외부 콘텐츠 읽기, 외부 전송이 한 세션에 모이나요? 모인다면 외부 전송을 승인 대상으로 묶습니다 |
| 격리와 허용 목록 | 확신이 없는 서버는 컨테이너에서 돌립니다. 조직은 허용한 서버 목록을 관리형 설정으로 배포할 수 있습니다 |

(출처: [MCP 문서, Local Server Security](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/local-server-security), [Claude Code 문서, Security](https://code.claude.com/docs/en/security#mcp-security), 2026-10 확인)

보안은 13.3절에서 한 번 더 나옵니다. 모델이 아무리 좋아져도 실행 단계의 승인이 남는 이유 가운데 하나가 이것입니다.

### 6.3 Subagent

Subagent는 **격리된 자기 컨텍스트에서 일하고 요약만 돌려주는** 일꾼입니다. 조사를 하려고 파일 40개를 읽으면 그 40개가 이후 모든 호출에 실립니다(1.4절). 같은 조사를 서브에이전트에 맡기면 읽은 파일은 서브에이전트의 컨텍스트에 남고, 메인 대화에는 결론 몇 줄만 돌아옵니다. 5.4절의 Skill과는 방향이 반대입니다.

| | Skill | Subagent |
| --- | --- | --- |
| 정체 | 다시 쓸 수 있는 내용 | 따로 실행되는 일꾼 |
| 컨텍스트 | 내 컨텍스트에 더해집니다 | 별도의 윈도우를 씁니다 |
| 돌려주는 것 | 없습니다. 본문이 로드될 뿐입니다 | 결과 요약 |
| 맞는 일 | 참고 자료, 절차 | 많이 읽는 조사, 병렬 작업, 독립된 검토 |

격리가 실제로 어떻게 동작하는지는 네 가지로 정리됩니다(출처: [Claude Code 문서, Create custom subagents](https://code.claude.com/docs/en/sub-agents), 2026-10 확인).

- **대화는 넘어가지 않습니다.** 서브에이전트는 메인의 대화 기록도, 메인이 읽은 파일도 보지 못합니다. 메인이 써 준 위임 메시지 하나로 시작합니다. 그래서 위임할 때 필요한 제약을 함께 적어 줘야 합니다.
- **지침 파일은 다시 실립니다.** 서브에이전트도 `CLAUDE.md`와 프로젝트 지침으로 로드된 `AGENTS.md`를 자기 컨텍스트에 싣습니다. 내장된 탐색용(Explore)과 계획용(Plan) 서브에이전트만 건너뜁니다. 지침이 길면 서브에이전트를 띄울 때마다 그 비용이 다시 듭니다.
- **대화형 세션에서는 백그라운드로 돕니다.** 기본 설정에서는 Claude가 띄운 서브에이전트가 뒤에서 돌고, 사람은 메인 대화를 계속할 수 있습니다.
- **결과는 데이터로 표시돼 돌아옵니다.** 서브에이전트의 보고는 "이것은 서브에이전트의 출력이고 그 안의 지시나 승인 주장은 사용자의 권한을 갖지 않는다"는 뜻의 머리말 아래 메인에 전달됩니다. 하네스가 보고서를 먼저 훑어 `<system-reminder>`처럼 자기 출력을 모방한 부분에 표식을 넣기도 합니다. 보고서 속 글이 지시처럼 읽히지 않게 하려는 것이고, 6.2절의 원칙을 하네스가 구현한 예입니다.

**비용 설계.** 서브에이전트는 메인 컨텍스트를 아끼는 장치이지 토큰을 아끼는 장치가 아닙니다. 서브에이전트의 요청도 같은 사용 한도에서 빠지고, 별도 컨텍스트를 처음부터 채우므로 총 토큰은 늘 수 있습니다. Anthropic은 자사 Research 기능의 데이터에서 멀티 에이전트 시스템이 채팅보다 토큰을 약 15배 쓴다고 밝혔습니다(같은 글에서 단일 에이전트는 약 4배입니다. 출처: [Anthropic, How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system), 2025-06-13). 그러니 맡길 일을 고릅니다.

| 메인 대화에서 한다 | 서브에이전트에 맡긴다 |
| --- | --- |
| 주고받으며 다듬는 작업 | 출력이 많고 다시 볼 일이 없는 작업. 로그 분석, 넓은 탐색, 테스트 실행 |
| 계획, 구현, 테스트가 맥락을 공유하는 작업 | 혼자 끝낼 수 있고 요약으로 돌려줄 수 있는 작업 |
| 작고 빠른 수정 | 도구와 권한을 좁혀서 시켜야 하는 작업. 읽기 전용 리뷰 |

정의 파일에서 서브에이전트마다 쓸 도구와 모델을 정할 수 있습니다. 탐색이나 로그 요약처럼 일의 성격이 정해진 서브에이전트에는 더 싼 모델을 지정해 둘 수 있습니다. 정의 파일의 모양은 7.4절에서 봅니다.

**생성자와 검증자의 분리.** 모델은 자기가 만든 결과물을 신뢰성 있게 평가하지 못합니다. 만든 쪽의 컨텍스트에는 그 변경에 이른 추론이 그대로 남아 있습니다. 그래서 검토는 새 컨텍스트의 서브에이전트에 맡깁니다. 검증자는 diff와 판정 기준만 받고, 그 변경을 만든 추론은 보지 못합니다. 2.3절의 추론적 센서가 이것입니다. 주의할 점도 문서에 적혀 있습니다. 빈틈을 찾으라고 시킨 리뷰어는 결과물이 멀쩡해도 대개 무언가를 보고합니다. 정확성과 요구사항에 영향을 주는 것만 지적하게 합니다(출처: [Claude Code 문서, Best practices](https://code.claude.com/docs/en/best-practices#add-an-adversarial-review-step), 2026-10 확인). 이 구조는 14.4절의 자동 리뷰에서 다시 쓰이고, 검증자도 같은 모델이라 같은 맹점을 공유한다는 한계는 14.8절에서 다룹니다.

### 6.4 Plugin

Plugin은 부품을 묶은 **배포 단위**입니다. Skill, 서브에이전트 정의, Hook, MCP 서버 설정을 한 디렉터리에 담아 한 번에 설치하고 한 번에 갱신합니다.

```text
team-embedded/
├─ .claude-plugin/plugin.json    이름, 버전, 설명
├─ skills/isr-review/SKILL.md    /team-embedded:isr-review 로 호출된다
├─ agents/pr-reviewer.md         서브에이전트 정의
├─ hooks/hooks.json              훅 설정
└─ .mcp.json                     MCP 서버 정의
```

만들 시점은 **두 번째 저장소가 같은 설정을 원할 때**입니다. 저장소 하나에서 나만 쓰는 동안에는 `.claude/` 아래에 그대로 두는 편이 낫습니다. 팀에 나눠 줄 때는 사내 Git 저장소를 마켓플레이스(플러그인 목록을 담은 카탈로그)로 등록해 이름으로 설치하게 합니다. 플러그인의 스킬에는 플러그인 이름이 앞에 붙으므로 로컬 스킬과 이름이 부딪치지 않습니다.

주의할 것은 두 가지입니다. 첫째, 설치한 플러그인은 내 PC에서 내 권한으로 코드를 실행할 수 있습니다. 훅과 MCP 서버가 들어 있으면 그것은 실행되는 코드이므로, 설치 전에 내용을 보고 변경도 코드 리뷰를 거칩니다. 둘째, 켜 둔 플러그인의 스킬과 서브에이전트 설명은 쓰지 않는 세션에서도 매 호출에 실립니다(출처: [Claude Code 문서, Plugins overview](https://code.claude.com/docs/en/plugins/overview), 2026-10 확인).

플러그인의 포장 형식은 도구마다 다릅니다. Skill과 MCP 서버를 묶는 공개 규격(Agent Plugins 1.0, 2026-08)이 나와 있지만, 2026년 10월 현재 Claude Code는 그 호환 목록에 없습니다(출처: [agent-plugins.org](https://agent-plugins.org/compatible-clients), 2026-10 확인). 그래서 지식은 Skill과 지침에 두고 포장은 얇게 유지합니다. 이식성은 3.5절에서 다뤘습니다.

### 핵심 정리

- MCP는 외부 시스템을 표준 방식으로 잇고 자격증명을 서버 쪽에 두지만, CLI로 되는 일과 검토하지 않은 서버에는 쓰지 않습니다.
- MCP 서버는 내 권한으로 도는 남의 코드이고 그 출력은 모델의 컨텍스트에 들어가므로, 출처와 버전과 실행 명령을 점검하고 도구가 읽어 온 것은 데이터로 다룹니다.
- Subagent는 메인 컨텍스트를 아끼는 대신 총 토큰을 더 쓰므로, 출력이 많은 조사와 독립된 검증처럼 격리의 값이 있는 일에 맡깁니다.

**다음 장으로**: 부품이 여섯 가지나 됩니다. 무엇부터, 어떤 기준으로 고르나요

---

## 7장. 부품 선택과 내 환경 셋업

- **핵심 질문**: 내 작업에는 어떤 부품 조합이 맞나요
- **도달 목표**: 필요 신호에 따라 부품을 하나씩 추가하는 순서를 정하고, 자기 환경의 첫 셋업을 마칩니다

6장은 "부품이 여섯 가지나 된다. 무엇부터, 어떤 기준으로 고르는가"로 끝났습니다. 이 장은 그 질문에 두 번 답합니다. 7.1\~7.4절은 가장 헷갈리는 세 부품인 Subagent, Skill, MCP를 가르는 기준을 줍니다. 7.5\~7.6절은 순서를 줍니다. 부품보다 먼저 익힐 기본 습관이 있고, 첫 셋업은 일곱 단계면 됩니다.

여섯 부품을 한꺼번에 만들 필요는 없습니다. 이 장의 요지는 **신호가 올 때 하나씩 더한다**는 것입니다.

### 7.1 누가 · 어떻게 · 무엇에

Subagent, Skill, MCP가 헷갈리는 데는 이유가 있습니다. 셋 다 결국 모델의 컨텍스트에 글을 넣고 실행할 능력을 붙이는 방법입니다. 그래서 무엇을 골라도 같은 목표를 구현할 수 있습니다. 다만 잘하는 자리가 다릅니다. 한 줄로 줄이면 Subagent는 두뇌, Skill은 매뉴얼, MCP는 손발입니다.

| 부품 | 비유 | 답하는 질문 |
| --- | --- | --- |
| Subagent | 별도의 두뇌 | **누가** 생각하나요? 격리된 컨텍스트에 맡깁니다 |
| Skill | 매뉴얼 | **어떻게** 하나요? 필요할 때 펼치는 절차 지식입니다 |
| MCP | 손발 | **무엇에** 닿나요? 시스템 경계 밖으로 나갑니다 |

세 부품의 차이를 항목별로 놓으면 다음과 같습니다.

| 항목 | Subagent | Skill | MCP |
| --- | --- | --- | --- |
| 정체 | 독립된 컨텍스트를 가진 별도의 에이전트 | 필요할 때 로드되는 지침과 딸린 파일 | 외부 능력을 표준 프로토콜로 내놓는 서버 |
| 메인 컨텍스트에 들어가는 것 | 결과 요약만 | 평소에는 이름과 설명, 쓸 때 본문 | 도구 정의와 호출 결과 (4.4절) |
| 실행 주체 | 새 에이전트 루프 | 호출한 에이전트 자신. 그 셸과 그 권한으로 실행합니다 | 별도 프로세스나 원격 서버 |
| 고유한 능력 | 컨텍스트 격리, 병렬 실행, 모델과 권한의 분리 | 지식의 재사용과 지연 로드 | 자격증명 격리, 강제 가드레일, 상태 유지 |
| 드는 비용 | 호출할 때마다 토큰과 지연 | 거의 없습니다 | 서버 운영. 배포, 인증, 버전 관리 |

### 7.2 남의 역할을 시킬 때의 대가

"무엇으로든 구현할 수 있다"는 말을 예로 확인합니다. 목표는 하나입니다. **PR을 가져와 체크리스트대로 리뷰하고 코멘트를 남깁니다.** 이것을 세 부품 가운데 하나만 써서 각각 구현할 수 있습니다.

| 방식 | 체크리스트를 두는 곳 | PR 조회와 코멘트 게시 | 동작은 하지만 치르는 대가 |
| --- | --- | --- | --- |
| (A) Subagent만 | 서브에이전트의 시스템 프롬프트 | 서브에이전트가 셸에서 CLI를 부릅니다 | 체크리스트가 그 서브에이전트 안에 갇힙니다. 메인이나 다른 서브에이전트가 다시 쓰지 못합니다 |
| (B) Skill만 | `SKILL.md` 본문 | 스킬에 딸린 스크립트가 API를 부릅니다 | diff 전체가 메인 컨텍스트에 쌓입니다. API 토큰이 에이전트의 환경 변수에 있어야 합니다 |
| (C) MCP만 | 서버의 Prompts 템플릿 | 서버의 Tools | 체크리스트를 고치려면 서버를 다시 배포해야 합니다. Prompts는 사용자가 골라야 쓰입니다 |

셋 다 동작합니다. 그러나 본업이 아닌 일을 맡은 칸에서는 대가를 치릅니다. 아홉 칸으로 펼치면 다음과 같습니다.

| 맡기는 일 → | 격리된 작업 수행 | 절차 지식 제공 | 외부 접근 |
| --- | --- | --- | --- |
| **Subagent** | ◎ 본업 | △ 지식이 그 에이전트에 갇힙니다. 쓸 때마다 새로 띄우는 비용이 듭니다 | ✕ 스스로는 닿을 수단이 없습니다. 셸이나 MCP를 빌려 씁니다 |
| **Skill** | △ 메인 컨텍스트에서 실행돼 중간 산출물이 쌓입니다 | ◎ 본업 | △ 토큰이 에이전트 환경에 노출됩니다. 규칙은 요청일 뿐 강제가 아닙니다 |
| **MCP** | △ 서버가 안에서 모델을 불러 처리할 수는 있지만, 추론 과정이 서버 안에 숨어 고치기 어렵습니다 | △ Prompts는 사용자가 고르는 구조이고 서버 설명은 항상 로드됩니다. 고치려면 다시 배포합니다 | ◎ 본업 |

대가는 세 가지로 모입니다.

- **컨텍스트 오염.** Skill로 격리해야 할 작업을 하면 중간 결과가 메인 대화에 쌓입니다.
- **지식 고립.** Subagent나 MCP 서버 안에 매뉴얼을 넣으면 다른 곳에서 다시 쓰기 어렵습니다.
- **보안 노출.** Skill로 외부에 연결하면 자격증명과 가드레일을 에이전트에게 맡기게 됩니다.

셋째 대가는 5.5절에서 본 요청과 강제의 차이와 같은 이야기입니다. 스킬에 "보호 브랜치로 가는 PR은 머지하지 않는다"고 적으면 요청입니다. 모델이 놓칠 수 있고, 읽어 온 글에 휘둘려 어길 수도 있습니다(6.2절). 같은 규칙을 MCP 서버의 머지 도구 안에 코드로 넣으면 강제입니다. 검사하는 쪽이 모델이 아니기 때문입니다. 코드는 7.4절에서 봅니다.

### 7.3 선택 신호와 기본값

고르는 방법은 이렇습니다. 만들려는 기능을 **누가, 어떻게, 무엇에**의 세 관심사로 쪼갠 뒤 각각을 맞는 부품에 배정합니다. 어느 부품이 필요한지는 아래 신호로 알아봅니다.

| 이런 신호가 보이면 | 고를 것 |
| --- | --- |
| 같은 절차, 사내 규칙, 판단 기준을 반복해서 설명합니다. 자주 쓰지는 않아 늘 실어 두기에는 아깝습니다. 내용이 자주 바뀝니다 | Skill |
| 자격증명을 에이전트에게 보이면 안 됩니다. 쓰기와 삭제에 우회할 수 없는 가드레일이 필요합니다. 여러 팀이 같은 연결을 함께 씁니다 | MCP |
| 중간 산출물이 커서 메인 컨텍스트를 채웁니다. 서로 독립된 작업을 병렬로 돌릴 수 있습니다. 더 좁은 권한으로 돌리고 싶습니다. 작성자와 분리된 검증이 필요합니다 | Subagent |

신호가 분명하지 않을 때의 **기본값**은 순서로 정해져 있습니다.

1. **Skill에서 시작합니다.** 비용이 가장 싸고, 매뉴얼은 어차피 Skill에 둡니다. 외부 연결도 처음에는 스킬에 딸린 스크립트로 충분합니다.
2. **접근 문제가 생기면 MCP를 더합니다.** 자격증명 격리, 강제 가드레일, 여러 팀의 공유가 필요해지는 시점입니다.
3. **컨텍스트, 병렬, 독립성 문제가 생기면 Subagent로 나눕니다.**

나중에 옮기기 쉽게 해 두는 요령이 하나 있습니다. 외부 연결 로직을 처음부터 스킬 본문과 분리해 둡니다. 그러면 스크립트에서 MCP 서버로 올릴 때 `SKILL.md`는 호출 방법 한 줄만 바뀝니다.

```text
pr-review/
├─ core/review_ops.py    실제 로직
├─ cli.py                1단계: Skill의 스크립트가 부르는 얇은 래퍼
├─ server.py             2단계: 필요해지면 더하는 MCP 래퍼
└─ skill/SKILL.md        매뉴얼
```

여섯 부품 전체로 넓혀도 원칙은 같습니다. 부품마다 "지금 만들어야 한다"는 신호가 있습니다.

| 신호 | 대응 | 관련 절 |
| --- | --- | --- |
| 같은 규약을 두 번 지적했습니다 | 지침 파일에 한 줄을 더합니다 | 5.2 |
| 같은 프롬프트나 절차를 세 번 입력했습니다 | Skill로 저장합니다 | 5.4 |
| 매번 자동으로 일어나야 합니다. 한 번도 어기면 안 됩니다 | Hook으로 만듭니다 | 5.5 |
| 다른 시스템의 화면에서 데이터를 복사해 붙이고 있습니다 | MCP 서버를 연결합니다 | 6.1 |
| 조사의 부산물이 대화를 어지럽힙니다 | Subagent로 격리합니다 | 6.3 |
| 두 번째 저장소가 같은 설정을 원합니다 | Plugin으로 묶습니다 | 6.4 |

(출처: [Claude Code 문서, Extend Claude Code](https://code.claude.com/docs/en/features-overview#build-your-setup-over-time)의 표를 이 세미나의 절 번호에 맞춰 옮겼습니다. 2026-10 확인)

같은 신호가 이미 만든 부품을 고칠 시점도 알려 줍니다. 리뷰에서 같은 코멘트가 반복된다면 그것은 한 번의 지적으로 끝낼 일이 아니라 지침을 고칠 일입니다. 2.4절의 제1원칙이 여기서 실제 절차가 됩니다.

### 7.4 역할대로 조합한 예

7.2절의 PR 리뷰를 이번에는 역할대로 나눕니다. 리뷰는 읽기 전용 서브에이전트가 하고, 기준은 Skill에 두고, 저장소 서버 접근은 가드레일을 가진 MCP 서버가 맡습니다. 저장소 서버는 사내 Bitbucket을 가정합니다.

**두뇌.** 서브에이전트 정의는 `.claude/agents/pr-reviewer.md`에 둡니다. 쓸 수 있는 도구가 읽기와 diff 조회뿐입니다.

```markdown
---
name: pr-reviewer
description: PR의 diff를 독립적으로 리뷰할 때 쓴다
tools: Read, Grep, Glob, mcp__bitbucket__get_pr_diff
model: sonnet
skills:
  - review-checklist
---

diff를 검토하고 지적 사항을 파일과 줄 번호와 함께 정리해 돌려준다. 코멘트를 직접 게시하지 않는다.
```

**매뉴얼.** 체크리스트는 `.claude/skills/review-checklist/SKILL.md`에 둡니다. 서브에이전트에는 `skills` 필드로 미리 실리고, 메인 에이전트도 필요할 때 같은 파일을 씁니다.

```markdown
---
name: review-checklist
description: 사내 C 코드 리뷰 기준이 필요할 때 쓴다
---

1. ISR 안전성: 블로킹 호출이 없는가, 공유 변수가 보호되는가
2. 오류 처리: 반환값을 무시하지 않는가, 타임아웃을 처리하는가
3. MISRA: 위반으로 보이면 [misra-deviations.md](misra-deviations.md)의 승인된 예외를 확인한다
```

**손발.** MCP 서버가 Bitbucket에 닿습니다. API 토큰은 이 서버 프로세스의 환경에만 있고, 규칙은 코드로 검사합니다.

```python
from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError

import bitbucket  # 사내 Bitbucket API를 감싼 클라이언트 모듈 (가상)

mcp = MCPServer("bitbucket")
PROTECTED_BRANCHES = {"main", "release"}


@mcp.tool()
def get_pr_diff(pr: int) -> str:
    """Get the diff of a pull request."""
    return bitbucket.get_diff(pr)


@mcp.tool()
def post_comment(pr: int, body: str) -> str:
    """Post a review comment on a pull request."""
    if len(body) > 10_000:
        raise ToolError("코멘트가 10,000자를 넘는다. 요약해서 다시 호출한다.")
    return bitbucket.post_comment(pr, body)


@mcp.tool()
def merge_pr(pr: int) -> str:
    """Merge a pull request."""
    if bitbucket.target_branch(pr) in PROTECTED_BRANCHES:
        raise ToolError("보호 브랜치로의 머지는 사람만 할 수 있다.")
    if not bitbucket.ci_passed(pr):
        raise ToolError("CI가 통과하지 않았다.")
    return bitbucket.merge(pr)
```

실행 흐름은 네 단계입니다.

1. 메인 에이전트가 `pr-reviewer` 서브에이전트에 리뷰를 맡깁니다.
2. 서브에이전트는 미리 실린 체크리스트를 기준으로, MCP 도구로 diff를 읽어 검토합니다.
3. 서브에이전트가 지적 사항을 요약해 돌려줍니다. 메인 컨텍스트에는 diff 전체가 아니라 요약만 남습니다.
4. 메인 에이전트가 사람의 확인을 받은 뒤 `post_comment`로 게시합니다.

이 조합에서 볼 것은 세 가지입니다. 서브에이전트에 쓰기 도구를 주지 않았으므로 **권한이 구조로 나뉩니다.** 체크리스트는 한 곳에 있고 **여러 에이전트가 다시 씁니다.** 보호 브랜치 규칙은 모델이 읽는 글이 아니라 **서버의 코드**이므로, 모델이 놓치든 인젝션에 휘둘리든 우회할 수 없습니다. 그리고 4번에 사람이 있습니다. 밖으로 나가는 행동 앞에 사람의 확인을 둔 것입니다(13.2절).

> **발표 전 확인**: MCP 서버 코드는 MCP Python SDK 2.3.0에서 가짜 `bitbucket` 모듈을 붙여 도구 호출과 오류 반환만 시험했습니다. 서브에이전트, Skill과 묶은 전체 흐름은 실제 세션에서 돌려 확인합니다. `model: sonnet`은 사내 백엔드가 제공하는 모델 이름에 맞춰 바꿉니다.

> **발표자 노트**: 예전 예제의 `from mcp.server.fastmcp import FastMCP`를 그대로 쓰면 SDK 2.x에서는 모듈을 찾지 못합니다. 2.x에서 `FastMCP`가 `MCPServer`로 이름이 바뀌었습니다. 모델이 읽고 고칠 수 있는 오류는 `ToolError`로 올립니다(출처: [MCP Python SDK 문서](https://py.sdk.modelcontextprotocol.io/), 2026-10 확인).

### 7.5 기본 사용 습관

부품을 하나도 만들지 않아도 오늘부터 쓰는 것이 세 가지 있습니다. 권한 모드, Plan mode, 세션 관리입니다. 부품보다 먼저 익힙니다. 이 셋은 Part 2에서 말할 사람의 개입 지점 가운데 **계획과 실행이 실제로 구현되는 자리**이기도 합니다(8.1절).

#### 권한 모드: 무엇을 묻고 무엇을 자동으로 실행할지

권한 모드는 에이전트가 사람에게 묻지 않고 할 수 있는 행동의 범위를 정합니다. Claude Code의 모드는 여섯 가지입니다.

| 모드 (설정값) | 화면의 이름 | 묻지 않고 실행하는 것 | 맞는 상황 |
| --- | --- | --- | --- |
| `default` | Manual | 읽기만 | 민감한 작업, 낯선 코드. 모든 행동을 직접 봅니다 |
| `acceptEdits` | accept edits | 읽기, 파일 편집, `mkdir`나 `mv` 같은 흔한 파일 명령 | 편집은 나중에 diff로 몰아서 볼 때 |
| `plan` | plan mode | 읽기와 탐색. 소스는 고치지 않습니다 | 바꾸기 전에 조사하고 계획을 받을 때 |
| `auto` | auto mode | 전부. 다만 분류기가 실행 전에 검토합니다 | 긴 작업, 승인 피로를 줄일 때 |
| `dontAsk` | don't ask | 미리 허용한 도구만. 나머지는 거부합니다 | 잠가 둔 CI와 스크립트 |
| `bypassPermissions` | bypass permissions | 전부. 검사가 없습니다 | 격리된 컨테이너와 VM 안에서만 |

(출처: [Claude Code 문서, Choose a permission mode](https://code.claude.com/docs/en/permission-modes), 2026-10 확인)

- **시작 모드.** v2.1.283부터 대화형 터미널과 VS Code 세션은 `auto`로 시작합니다. 세션 중에는 `Shift+Tab`으로 바꿉니다. `auto`에서 누르면 Manual, accept edits, plan mode 순으로 돕니다.
- **auto 모드의 동작.** 사람 대신 **분류기**라는 별도의 모델이 행동을 실행 전에 검토합니다. 요청 범위를 넘는 행동, 모르는 인프라를 건드리는 행동, 읽어 온 글에 휘둘린 것으로 보이는 행동을 막습니다. 기본으로 막는 것에는 내려받은 코드의 즉시 실행, 민감한 데이터의 외부 전송, 운영 배포와 마이그레이션, 강제 push, 세션 전부터 있던 파일을 되돌릴 수 없게 지우는 행동이 있습니다. 읽기와 작업 디렉터리 안의 편집은 분류기를 거치지 않습니다. 문서는 "Auto mode reduces permission prompts but does not guarantee safety."(auto 모드는 승인 프롬프트를 줄일 뿐 안전을 보장하지 않는다)라고 경고합니다.
- **사내 환경.** 사내 환경에서도 auto 모드를 쓸 수 있습니다(사내 확인). 다만 공짜가 아닙니다. 분류기도 모델 호출이고, 게이트웨이를 거치는 연결에서는 이 호출이 토큰 사용량에 포함돼 과금됩니다. 부담은 주로 셸 명령과 네트워크 작업에서 생깁니다.

> **발표 전 확인**: 사내 환경에서 분류기가 어떤 모델로 도는지, 같은 작업을 Manual과 auto로 했을 때 크레딧이 얼마나 다른지 대시보드로 확인합니다. 세션에서 `/status`를 실행하면 `Auto mode server` 행에서 서버 쪽 검토가 켜져 있는지 볼 수 있습니다.

모드는 바탕이고, 그 위에 **권한 규칙**을 얹습니다. 규칙은 허용(allow), 묻기(ask), 거부(deny)의 세 종류이고 거부, 묻기, 허용 순으로 평가됩니다. 거부 규칙은 어느 모드에서나 적용됩니다.

```json
{
  "permissions": {
    "allow": ["Bash(./run_tests.sh *)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(./secrets/**)"]
  }
}
```

이 규칙은 모델이 아니라 하네스가 집행합니다. 문서의 표현으로는 "Permission rules are enforced by Claude Code, not by the model."입니다. 지침에 적은 "push하지 않는다"는 요청이고 규칙은 강제입니다. 5.5절에서 본 구도와 같습니다. 규칙이 필요한 이유는 또 있습니다. 매번 묻는 Manual 모드가 가장 안전해 보이지만, 문서가 짚듯 열 번째 승인쯤에는 검토하지 않고 누르게 됩니다. 자주 쓰는 안전한 명령은 허용에 넣어 묻는 횟수를 줄이고, 사람의 주의는 되돌릴 수 없는 행동에 남겨 둡니다(13.2절).

#### Plan mode: 구현 전에 계획을 받아 승인한다

Plan mode에서 에이전트는 파일을 읽고 탐색하고 계획을 쓰지만 소스는 고치지 않습니다. 계획을 승인해야 편집이 시작됩니다. `Shift+Tab`으로 들어가거나, 프롬프트 앞에 `/plan`을 붙이거나, `claude --permission-mode plan`으로 시작합니다. 계획이 나오면 승인하고 진행하기, 승인하되 편집마다 직접 보기, 계획을 더 다듬기 가운데 하나를 고릅니다. `Ctrl+G`를 누르면 계획을 편집기에서 직접 고칠 수 있습니다.

계획 단계가 값진 이유는 **틀린 방향을 가장 싸게 잡는 지점**이기 때문입니다. 코드가 수백 줄 나온 뒤에 "이 방향이 아니다"를 발견하면 그 줄을 읽는 시간과 다시 만드는 토큰이 듭니다. 계획에서 발견하면 문단 하나를 고치면 됩니다. 표준 작업 흐름은 다음과 같습니다.

```text
탐색 → 계획(사람 승인) → 구현 → 자동 검증 → 사람 리뷰 → 커밋
```

모든 작업에 계획이 필요한 것은 아닙니다. 문서의 기준은 "If you could describe the diff in one sentence, skip the plan."(바뀔 내용을 한 문장으로 말할 수 있으면 계획을 건너뛴다)입니다. 접근이 불확실할 때, 여러 파일을 고칠 때, 낯선 코드를 만질 때 씁니다(출처: [Claude Code 문서, Best practices](https://code.claude.com/docs/en/best-practices#explore-first-then-plan-then-code), 2026-10 확인). 한 가지 주의가 있습니다. bypass permissions를 쓸 수 있게 시작한 대화형 터미널 세션에서는 승인 전의 편집 차단이 강제되지 않습니다.

#### 세션 관리: 한 세션은 한 목적

세션은 길어질수록 비싸지고(1.4절) 흐려집니다(1.3절). 그래서 세션을 작업의 단위로 다룹니다.

| 상황 | 할 일 | 명령 |
| --- | --- | --- |
| 관계없는 새 작업을 시작합니다 | 컨텍스트를 비웁니다 | `/clear` 또는 새 세션 |
| 같은 문제를 두 번 넘게 바로잡았습니다 | 실패한 시도가 쌓인 컨텍스트를 버리고, 배운 것을 담은 더 구체적인 요청으로 다시 시작합니다 | `/clear` |
| 맥락을 이어 가야 하는 긴 작업입니다 | 남길 것을 지정해 압축합니다. 계획과 결정은 파일로 남깁니다 | `/compact <남길 것>` |
| 방금 한 시도를 되돌리고 싶습니다 | 대화와 파일을 이전 지점으로 되돌립니다 | `Esc` 두 번 또는 `/rewind` |
| 내일 이어서 합니다 | 세션에 이름을 붙여 두고 다시 엽니다 | `/rename`, `claude --resume` |

(출처: [Claude Code 문서, Best practices](https://code.claude.com/docs/en/best-practices#manage-your-session), 2026-10 확인)

두 가지를 덧붙입니다. 컨텍스트가 오염됐다고 느끼면 압축보다 새 세션이 낫습니다. 압축은 무엇이 남을지를 내가 정하지 못하고, 새 세션은 필요한 결론만 골라 넘길 수 있습니다. 계획 문서를 파일로 남겨 두면 세션이 끊겨도 이어 갈 수 있습니다. 그리고 `/rewind`는 git을 대신하지 못합니다. 에이전트의 편집 도구로 바꾼 것만 되돌리고, 셸 명령으로 바꾼 것은 되돌리지 못합니다. 자주 커밋해서 언제든 돌아갈 수 있는 상태를 유지합니다.

세 습관을 Part 2의 말로 옮기면 다음과 같습니다.

| 습관 | 사람이 하는 일 | 개입 지점 (8.1절) |
| --- | --- | --- |
| 세션 관리 | 요청을 목적 단위로 끊고, 배운 것을 담아 다시 씁니다 | 요청 |
| Plan mode | 구현 전에 접근을 승인합니다 | 계획 |
| 권한 모드와 권한 규칙 | 위험한 행동을 허가하거나 막습니다 | 실행 |

검증은 5.5절의 Hook과 6.3절의 리뷰 서브에이전트, 그리고 사람의 리뷰가 맡습니다. Copilot CLI, Codex CLI, opencode에도 계획 모드와 권한 설정이 있습니다. 이름과 단계는 도구마다 다르므로 3.2절의 비교를 참고합니다.

### 7.6 내 환경 셋업 체크리스트

지금까지의 내용을 첫 셋업 순서로 줄이면 일곱 단계입니다. 앞 단계가 뒤 단계의 전제이므로 위에서부터 합니다.

| 순서 | 할 일 | 다 됐는지 확인하는 법 | 관련 절 |
| --- | --- | --- | --- |
| 1 | 도구를 고릅니다 | 내 조건 점검표로, 고른 이유를 한 줄로 말할 수 있습니다 | 3.1, 3.2, 3.6 |
| 2 | 백엔드에 연결합니다 | 새 세션에서 간단한 질문에 답이 옵니다. `claude --version`이 v2.1.283 이상입니다 | 3.5, 부록 A |
| 3 | 권한을 설정합니다 | 시작 모드를 정했습니다. 자주 쓰는 빌드와 테스트 명령은 허용에, 되돌릴 수 없는 명령은 묻기나 거부에 있습니다 | 7.5 |
| 4 | 지침 파일을 만듭니다 (단일 원본) | `AGENTS.md` 하나에 명령, 함정, 판단 기준이 200줄 아래로 적혀 있습니다 | 5.2, 5.3, 3.4 |
| 5 | `/context`로 확인합니다 | `Memory files`에 지침이 보입니다. 가장 큰 행이 무엇인지 압니다 | 1.5 |
| 6 | tool search를 설정합니다 | `ENABLE_TOOL_SEARCH`를 설정하기 전과 후의 `MCP tools` 행을 비교했습니다 | 4.8 |
| 7 | 첫 Hook을 겁니다 | 일부러 틀린 코드를 쓰게 해서, 편집 후 검사가 실제로 걸리는 것을 봤습니다 | 5.5 |

상세 절차는 부록 A에 있습니다. 설치, 백엔드 연결, tool search 설정값, 자주 만나는 오류를 다룹니다. 2번의 버전은 이 문서가 설명한 동작의 기준입니다. `AGENTS.md`를 직접 읽는 기능(사내 환경은 v2.1.281)과 `auto` 시작 모드(v2.1.283)가 그 버전에서 갖춰집니다.

이미 지침 파일이 있는 저장소라면 4번은 "만든다"가 아니라 "줄인다"가 됩니다. 구세대 모델에 맞춰 쌓인 줄을 감사하고 한 묶음씩 덜어 내는 절차는 5.2절에 있습니다.

순서에는 이유가 있습니다. 1\~3번은 도구가 안전하게 돌게 하는 일입니다. 4번은 프로젝트를 알려 주는 일입니다. 5\~6번은 매 호출에 실리는 고정 비용을 확인하고 줄이는 일입니다. 연결해 두고 쓰지 않는 MCP 서버가 있으면 세션을 시작하기 전에 끕니다(4.10절). 7번은 첫 강제 장치입니다. 여기까지가 첫 셋업입니다.

Skill, MCP, Subagent, Plugin은 이 표에 없습니다. 빠뜨린 것이 아닙니다. 이 넷은 7.3절의 신호가 올 때 하나씩 더합니다. 그리고 이 표는 한 번 채우고 끝나는 것이 아닙니다. 도구의 버전, 승인된 백엔드, 과금 방식이 바뀌면 1번부터 다시 봅니다(3.6절).

> **질문**: 일곱 줄 가운데 지금 내 환경에서 이미 된 것은 몇 줄인가요? 안 된 줄 가운데 오늘 할 수 있는 것은 무엇인가요?

### 7.7 Part 1 정리

Part 1의 일곱 장을 한 문장씩으로 줄이면 다음과 같습니다.

| 장 | 한 문장 |
| --- | --- |
| 1장 | 에이전트는 모델 추론과 도구 실행의 반복이고, 모델은 컨텍스트에 있는 것만 압니다 |
| 2장 | 같은 모델의 결과를 가르는 것은 하네스이고, 우리가 만드는 것은 가이드와 센서로 채운 사용자 하네스입니다 |
| 3장 | 사용률은 조달이 정하고 선호도는 하네스가 정합니다. 언제든 옮길 수 있게 지식을 지침과 Skill에 둡니다 |
| 4장 | 비용은 판단이 필요 없는 고정 영역부터 줄이고, 그 첫 대상이 MCP와 tool search입니다 |
| 5장 | 지침과 Skill은 요청이고 Hook은 강제이며, 지침은 필요할 때만 읽히게 합니다 |
| 6장 | 프로젝트의 경계를 넘는 부품에는 보안과 토큰이라는 대가가 따릅니다 |
| 7장 | 부품은 신호가 올 때 하나씩 더하고, 그보다 먼저 권한 모드와 Plan mode와 세션 관리를 익힙니다 |

일곱 장을 꿰는 개념은 둘입니다. 하나는 **컨텍스트 윈도우**입니다. 무엇이 실리고 무엇이 빠지는지가 비용과 품질을 함께 정했습니다. 다른 하나는 **검증 장치(센서)** 입니다. 요청으로 알려 준 것을 컴파일, 테스트, 훅, 리뷰로 확인하는 구조가 하네스의 절반이었습니다.

이제 질문 하나가 남습니다. 지침을 쓰고, 훅을 걸고, 권한을 설정하고, 비용도 줄였습니다. **환경을 다 갖추면 전부 맡겨도 되나요?**

Part 2의 답은 "아니다"이고, 이유는 셋업이 모자라서가 아닙니다. Part 1의 두 개념이 그대로 한계의 출발점이 됩니다. 컨텍스트 윈도우가 유한하다는 사실은 에이전트가 코드베이스를 전부 볼 수 없다는 한계로 이어집니다(9장). 센서로 확인되는 것만 믿을 수 있다는 사실은 모델이 채점되는 것만 잘한다는 한계와 뿌리가 같습니다(11장). 7.5절에서 Plan mode와 권한 모드를 익히면서 우리는 사람이 설 자리를 이미 두 곳 만들었습니다. 그 자리가 왜 필요하고, 어디에 더 있어야 하는지가 Part 2의 주제입니다.

### 핵심 정리

- Subagent는 누가, Skill은 어떻게, MCP는 무엇에를 맡고, 본업이 아닌 일을 시키면 컨텍스트 오염, 지식 고립, 보안 노출 가운데 하나를 치릅니다.
- 부품은 Skill에서 시작해 신호가 올 때 하나씩 더하고, 그보다 먼저 권한 모드, Plan mode, 세션 관리를 익힙니다.
- 첫 셋업은 도구 선택에서 첫 Hook까지 일곱 단계이고, 조건이 바뀌면 처음부터 다시 점검합니다.

**다음 장으로**: 환경을 다 갖추면 전부 맡겨도 되나요
