# MCP가 많을수록 토큰이 얼마나 더 드는가: 원인과 해결 (Claude Opus 5.5 기준)

> 작성일: 2026-10-11
> 대상: Claude Code 등 코딩 에이전트에 MCP 서버를 여러 개 연결해 쓰는 개발자
> 전제: 2026년 6월부터 GitHub Copilot도 토큰 기반 과금(GitHub AI Credits, 1 credit = $0.01)으로 바뀌었으므로, 아래 계산은 Anthropic API 직접 사용과 Copilot API 경유 환경 모두에 적용된다.

---

## 0. 요약

- 코딩 에이전트는 작업 하나를 처리하면서 API를 수십 번 호출하고, **호출할 때마다 시스템 프롬프트와 MCP 도구 정의를 포함한 전체 컨텍스트를 다시 보낸다.**
- MCP 도구 정의는 "사용하지 않아도" 매 호출에 실린다. 그래서 MCP를 추가할수록 **대기 상태의 고정 비용**이 호출 횟수만큼 곱해진다.
- prompt caching으로 반복 부분은 싸게(캐시 읽기) 처리되지만, 캐시는 **마지막 호출 후 5분**이 지나면 만료된다. 사용자가 다음 요청을 5분 넘게 고민하거나, 빌드·테스트가 5분 넘게 걸리면 고정 영역 전체를 다시 비싼 단가(캐시 쓰기)로 기록한다.
- Opus 5.5 기준 계산 결과, 기본 컨텍스트가 4%인 세션과 12%인 세션은 **작업당 약 $0.83(+21%) 차이**가 나고, 월 $600 예산으로 처리할 수 있는 작업이 약 153건 → 126건으로 줄어든다.
- 근본 해결은 MCP 서버를 컨텍스트 효율적으로(3계층) 설계하는 것이지만, 대부분 남이 만든 MCP를 쓰므로 현실적이지 않다.
- 대신 **클라이언트 쪽 defer loading(tool search)** 을 켜면, 도구 이름만 미리 싣고 필요한 도구만 그때 불러온다. 같은 계산에서 12% 세션의 비용이 4% 세션 수준(약 $3.96\~$4.00)으로 내려간다.

---

## 1. 코딩 에이전트의 loop 구조: 왜 매번 다시 보내는가

LLM API는 상태를 저장하지 않는다(stateless). 그래서 에이전트는 도구를 한 번 쓸 때마다 **처음부터 지금까지의 모든 내용**을 다시 보낸다.

```python
# 코딩 에이전트 loop를 단순화한 모습
messages = [user_request]
while True:
    resp = client.messages.create(
        model="claude-opus-5-5",
        system=SYSTEM_PROMPT,          # 매 호출마다 전송
        tools=BUILTIN_TOOLS + MCP_TOOLS,  # ← MCP 도구 정의도 매 호출마다 전송
        messages=messages,             # 지금까지 쌓인 대화 전체
    )
    if resp.stop_reason != "tool_use":
        break                          # 최종 답변 → 작업 종료
    result = run_tool(resp)            # Read, Bash, MCP 도구 등 실행
    messages += [resp, result]         # 대화가 계속 커짐
```

작업 하나에 도구 호출이 40번 일어나면 API 호출도 40번이고, `SYSTEM_PROMPT + MCP_TOOLS`는 **40번 input 토큰으로 계산**된다.

```
호출 1 : [system][tools: builtin + MCP][요청]
호출 2 : [system][tools: builtin + MCP][요청][응답1][도구결과1]
호출 3 : [system][tools: builtin + MCP][요청][응답1][도구결과1][응답2][도구결과2]
 ...
호출 40: [system][tools: builtin + MCP][ ......... 지금까지 전부 ......... ]
         └──────── 고정 영역 ────────┘
```

### 1.1 고정 영역(기본 context)을 구성하는 요소

세션을 새로 열고 아무 요청도 하지 않은 상태에서 이미 차지하고 있는 컨텍스트가 "기본 context"다. Claude Code 기준으로 다음 요소들로 이루어진다. `/context` 명령을 실행하면 항목별 크기를 직접 볼 수 있다.

| 요소 | 무엇이 실리나 | 크기를 결정하는 것 |
|---|---|---|
| 시스템 프롬프트 | 에이전트의 기본 동작 지침 | 도구 버전에 따라 고정. 사용자가 조절할 수 없음 |
| Built-in 도구 | Read, Edit, Bash, Grep 등의 정의(이름 + 설명 + 스키마) | 고정. 소수의 도구를 공급자가 직접 다듬어 둠 |
| Memory | `CLAUDE.md`(사용자, 프로젝트, 하위 폴더), auto memory 파일 | **사용자가 쓴 텍스트 분량**. 내용 전체가 실림 |
| Skills | 설치된 skill마다 **이름과 description(frontmatter)만** | skill 개수. 본문(SKILL.md)은 호출될 때만 로딩 |
| Custom agents | 서브에이전트 정의의 이름과 설명 | 정의 개수 |
| MCP 도구 | 연결된 모든 서버의 **모든 도구 정의 전체** + 서버 instructions | 서버 수 × 서버당 도구 수 × 도구당 스키마 크기 |

```bash
claude
> /context
# 예시 출력 형태 (값은 환경마다 다름)
#   System prompt     …
#   System tools      …   ← built-in 도구
#   MCP tools         …   ← 보통 이 항목이 가장 큼
#   Custom agents     …
#   Memory files      …   ← CLAUDE.md 등
#   Skills            …   ← 이름과 description만
#   Messages          …   ← 대화(작업하면서 늘어나는 부분)
```

이 중 MCP가 왜 단연 큰지는 3장에서 설명한다.

---

## 2. Prompt caching과 5분 규칙

### 2.1 캐시가 비용을 어떻게 바꾸는가

같은 앞부분(prefix)이 반복되면 API가 캐시해 두고 할인된 단가로 계산한다.

| 구분 | Opus 5.5 단가 (100만 토큰당) | 언제 발생하나 |
|---|---|---|
| 일반 입력 | $4.00 | 캐시를 쓰지 않을 때 |
| 캐시 쓰기 | $5.00 | 처음 보내는 부분, 또는 캐시가 만료된 뒤 다시 보낼 때 |
| 캐시 읽기 | $0.20 | 5분 안에 같은 prefix를 다시 보낼 때 |
| 출력 | $20.00 | 모델이 생성한 토큰 |

캐시 읽기가 쓰기보다 **25배** 싸기 때문에, 캐시가 유지되느냐가 비용을 크게 좌우한다.

### 2.2 5분의 기준

- 기준은 **마지막으로 그 캐시를 사용한 API 호출 시점**이다. 캐시가 사용될 때마다 5분이 다시 시작된다.
- "작업 시간"은 상관없다. 20분짜리 작업도 그 안에서 호출이 계속 이어지면 캐시는 살아 있다.
- 작업의 마지막 단계는 모델이 최종 답변을 만드는 API 호출이므로, 실질적으로 **작업이 끝난 시점부터 5분**이다.

### 2.3 캐시가 깨지는 두 가지 상황

```
① 사용자 입력 간격이 5분 초과
   [작업 A ... call30(최종 답변)] ── 사용자가 결과 검토 7분 ──→ [작업 B call1]
                                                                ↑ 캐시 만료 → 전체 prefix 캐시 쓰기($5)

② 작업 중 도구 실행이 5분 초과
   call12 → [make build 7분] → call13
                                ↑ 캐시 만료 → 그 시점까지의 prefix 전체 캐시 쓰기($5)
```

```python
TTL = 300  # 초

def simulate(call_times):
    last_use = None
    for t in call_times:
        status = "WRITE ($5.00/M)" if last_use is None or t - last_use > TTL else "READ  ($0.20/M)"
        print(f"t={t:>4}s  {status}")
        last_use = t

# 짧은 도구 3번 → 7분짜리 빌드 → 사용자가 6분 고민 후 다음 요청
simulate([0, 20, 45, 465, 480, 840])
# t=   0s  WRITE   ← 첫 호출
# t=  20s  READ
# t=  45s  READ
# t= 465s  WRITE   ← 빌드 7분 동안 만료
# t= 480s  READ
# t= 840s  WRITE   ← 사용자 대기 6분 동안 만료
```

실제 개발 업무에서는 결과를 읽고, 코드를 직접 확인하고, 회의에 다녀오는 일이 흔하므로 **사용자 입력 간격이 5분을 넘는 경우가 드물지 않다.** 이때 MCP 도구 정의를 포함한 고정 영역 전체가 다시 비싼 단가로 기록된다.

---

## 3. 왜 MCP는 대기 상태에서 컨텍스트를 많이 차지하는가

### 3.1 도구 정의 전체가 매 요청에 실린다

MCP 서버가 연결되면 클라이언트는 서버와 두 번의 교환을 한다.

**① `initialize` 응답: 서버 이름과 서버 instructions**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "serverInfo": { "name": "jira", "version": "1.4.0" },
    "capabilities": { "tools": { "listChanged": true } },
    "instructions": "Tools for Atlassian Jira. Use them when the user asks to look up, create, update, assign or transition Jira issues, or to query sprints and boards. Issue keys look like PLAT-123."
  }
}
```

**② `tools/list` 응답: 도구 정의 전체**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "search_issues",
        "description": "Search Jira issues using JQL. Supports filtering by project, assignee, status, sprint, labels, fix version and custom fields. Returns key, summary, status, assignee and updated time. Use startAt/maxResults for pagination...",
        "inputSchema": {
          "type": "object",
          "properties": {
            "jql":        { "type": "string",  "description": "JQL query string, e.g. project = PLAT AND status = 'In Progress'" },
            "fields":     { "type": "array",   "items": { "type": "string" }, "description": "Fields to return ..." },
            "startAt":    { "type": "integer", "description": "Index of the first result ..." },
            "maxResults": { "type": "integer", "description": "Max results (1-100) ..." },
            "expand":     { "type": "string",  "enum": ["renderedFields", "names", "schema", "changelog"] }
          },
          "required": ["jql"]
        }
      },
      { "name": "get_issue",        "description": "...", "inputSchema": { "...": "..." } },
      { "name": "create_issue",     "description": "...", "inputSchema": { "...": "..." } },
      { "name": "transition_issue", "description": "...", "inputSchema": { "...": "..." } }
      // ... 서버에 따라 수십 개
    ]
  }
}
```

이런 도구 하나가 대략 300\~800 토큰이고, 서버당 수십 개씩 있다.

**모델이 실제로 받는 컨텍스트: defer 끔 vs 켬**

클라이언트는 서버 이름을 붙여 도구 이름을 `mcp__<서버>__<도구>` 형태로 바꾼 뒤 모델에 전달한다. defer loading 여부에 따라 모델이 받는 내용이 달라진다.

```jsonc
// (A) defer 끔: 모든 도구의 전체 정의가 매 요청 tools 배열에 실림
{
  "tools": [
    { "name": "Read",  "description": "...", "input_schema": { /* built-in */ } },
    { "name": "Bash",  "description": "...", "input_schema": { /* built-in */ } },
    { "name": "mcp__jira__search_issues",     "description": "Search Jira issues using JQL ...", "input_schema": { /* 위 스키마 전체 */ } },
    { "name": "mcp__jira__get_issue",         "description": "...", "input_schema": { /* 전체 */ } },
    { "name": "mcp__jira__create_issue",      "description": "...", "input_schema": { /* 전체 */ } },
    { "name": "mcp__confluence__search_pages","description": "...", "input_schema": { /* 전체 */ } }
    // ... 연결된 모든 서버의 모든 도구
  ]
}
```

```jsonc
// (B) defer 켬: 도구 "이름"과 서버 instructions만 텍스트로 실리고,
//     전체 정의는 검색 도구(ToolSearch)를 통해 필요할 때만 불러옴
{
  "tools": [
    { "name": "Read",       "description": "...", "input_schema": { /* built-in */ } },
    { "name": "Bash",       "description": "...", "input_schema": { /* built-in */ } },
    { "name": "ToolSearch", "description": "Fetch full definitions of deferred tools ...", "input_schema": { /* query */ } }
  ],
  "system/context": [
    "Deferred tools (call ToolSearch to load before use):",
    "  mcp__jira__search_issues, mcp__jira__get_issue, mcp__jira__create_issue, mcp__jira__transition_issue,",
    "  mcp__confluence__search_pages, mcp__confluence__get_page, ...",
    "Server instructions:",
    "  [jira] Tools for Atlassian Jira. Use them when the user asks to look up, create, update, assign or transition Jira issues ...",
    "  [confluence] Tools for Confluence wiki pages. Use them to search, read or create pages ..."
  ]
}
```

(B)에서 모델은 **도구 이름(서버 이름 포함)과 서버 instructions만 보고** "이 요청에 Jira 도구가 필요하겠다"고 추론한 뒤 `ToolSearch`를 호출한다. 그래서 이름과 instructions의 품질이 검색 여부를 좌우한다. (위 (B)의 `system/context` 필드는 실제 API 필드명이 아니라, 모델 컨텍스트에 텍스트로 들어가는 내용을 보여주기 위한 표기다.)

### 3.2 왜 MCP가 다른 요소보다 단연 큰가

1.1의 요소들과 비교하면 MCP만 구조적으로 다르다.

| 요소 | 로딩 방식 | 누가 크기를 통제하나 | 증가 방식 |
|---|---|---|---|
| 시스템 프롬프트, built-in 도구 | 전체 | 에이전트 공급자가 최소화해 둠 | 고정 |
| Memory (CLAUDE.md) | 전체 | 사용자가 직접 씀 | 사용자가 쓴 만큼 |
| Skills | **이름과 description만** (본문은 호출 시) | skill 작성자, 단 본문은 기본 비용 아님 | skill 1개당 수십\~백여 토큰 |
| **MCP** | **모든 도구의 전체 스키마** | **서드파티 서버 작성자** | 서버 수 × 도구 수 × 스키마 크기 |

1. **로딩 단위가 가장 크다.** Skills는 처음부터 "목차(이름 + 한 줄 설명)만 싣고 본문은 필요할 때 읽는" progressive disclosure 구조다. 반면 MCP는 프로토콜상 `tools/list`가 모든 도구의 전체 스키마를 한 번에 돌려주고, 클라이언트는 이를 그대로 모델에 넘긴다(defer loading이 없다면).
2. **곱셈으로 늘어난다.** 서버 하나를 추가하면 도구가 수십 개씩 한꺼번에 들어온다. Skill 하나를 추가하는 것과 단위가 다르다.
3. **스키마는 토큰을 많이 쓰는 형식이다.** JSON의 중괄호, 키 이름, 타입, enum, 파라미터마다 붙는 설명이 모두 토큰이다. 같은 정보를 산문으로 쓸 때보다 길다.
4. **사용자가 줄일 수 없다.** CLAUDE.md는 내가 편집할 수 있지만, 외부 MCP 서버의 도구 description과 스키마는 서버 작성자가 정한다. 그리고 많은 서버가 컨텍스트 비용보다 "API 기능을 빠짐없이 노출"하는 데 맞춰 만들어져 있다.
5. **사용 여부와 무관하게 실린다.** Memory는 대개 모든 작업에 관련이 있지만, MCP 도구는 대부분의 작업에서 몇 개만 쓰이거나 아예 쓰이지 않는다. 비용 대비 활용도가 가장 낮다.

### 3.3 정의가 커지는 이유 (서버 설계 측면)

| 원인 | 설명 |
|---|---|
| API 엔드포인트 = 도구 1개 | 많은 MCP 서버가 기존 REST API를 그대로 감싸서, 엔드포인트마다 도구를 하나씩 만든다. GitHub, Jira, Slack 같은 서버는 도구가 수십\~백 개를 넘는다 |
| 상세한 description | 모델이 정확히 고르도록 설명을 길게 쓴다. 이 자체는 좋은 일이지만 토큰을 차지한다 |
| 복잡한 JSON Schema | 중첩 객체, enum, 파라미터별 설명이 모두 토큰이다 |
| 서버 instructions | 서버가 제공하는 사용 안내문도 함께 실린다 |
| 사용 여부와 무관 | 이번 작업에서 Slack을 전혀 안 써도 Slack 도구 정의는 매 호출 실린다 |

참고 수치:

- Anthropic API 문서: GitHub, Slack, Sentry, Grafana, Splunk를 함께 쓰는 구성은 작업 시작 전에 **약 55K 토큰**을 도구 정의로 소비할 수 있다.
- Agent SDK 문서: 도구 50개는 **10\~20K 토큰**을 쓴다. 또 도구가 30\~50개를 넘으면 **도구 선택 정확도도 떨어진다.**

즉 MCP가 많아질수록 "아무것도 안 하는데 드는 비용"과 "잘못된 도구를 고를 위험"이 함께 커진다.

---

## 4. 비용 계산 (Opus 5.5)

### 4.1 가정

| 항목 | 값 |
|---|---|
| A: 필수 설정만 (built-in 도구 + skill + 필수 MCP) | 기본 컨텍스트 40K (1M의 4%) |
| B: A + "유용해 보여서 추가한" MCP | 기본 컨텍스트 120K (1M의 12%) |
| 작업 중 컨텍스트 증가량 | 260K (A 기준 작업 종료 시 30%) |
| 작업 1건당 API 호출 수 | 40회 |
| 작업 1건당 출력 | 60K |
| 루프 내부 | 직전까지의 prefix는 캐시 읽기, 새로 붙는 부분은 캐시 쓰기 |
| 작업 시작 시 캐시 상태 | 50%는 5분 이내에 이어서 요청(캐시 유효), 50%는 5분 초과(캐시 만료) |

계산 모델:

```python
P_READ, P_WRITE, P_OUT = 0.20, 5.00, 20.00   # $/1M tokens
GROWTH, CALLS, OUT = 260_000, 40, 60_000

def task_cost(base, cold_start, long_tool_misses=0):
    step = GROWTH / CALLS
    reads = writes = 0.0
    for i in range(1, CALLS + 1):
        prefix = base + (i - 1) * step               # 이번 호출 직전까지의 컨텍스트
        if i == 1:
            if cold_start: writes += base            # 5분 초과 → 고정 영역 다시 쓰기
            else:          reads  += base            # 5분 이내 → 캐시 읽기
        elif i <= 1 + long_tool_misses:
            writes += prefix                         # 5분 넘는 도구 실행 → prefix 전체 다시 쓰기
        else:
            reads += prefix
    writes += GROWTH                                 # 새로 붙는 부분은 한 번씩 쓰기
    return (reads * P_READ + writes * P_WRITE + OUT * P_OUT) / 1e6

def avg(base, cold_ratio, long_tool_misses=0):
    return (cold_ratio * task_cost(base, True, long_tool_misses)
            + (1 - cold_ratio) * task_cost(base, False, long_tool_misses))

a, b = avg(40_000, 0.5), avg(120_000, 0.5)
print(f"A ${a:.2f} / B ${b:.2f} / 차이 ${b-a:.2f}")   # A $3.93 / B $4.76 / 차이 $0.83
```

### 4.2 기본 결과

| 기본 컨텍스트 | 작업 1건당 평균 비용 |
|---|---|
| 4% (40K) | $3.93 |
| 8% (80K) | $4.35 |
| 12% (120K) | $4.76 |
| 20% (200K) | $5.59 |
| 30% (300K) | $6.63 |

**기본 컨텍스트 1M의 1%(10K)마다 작업당 약 $0.10씩 늘어난다.** MCP를 계속 추가하면 비용이 선형으로 증가한다.

### 4.3 사용자 입력 간격(캐시 만료 비율)의 영향

| 다음 요청까지 5분 초과 비율 | A (4%) | B (12%) | 차이 | $600 기준 처리 작업 수 (A → B) |
|---|---|---|---|---|
| 0% (항상 바로 이어서 요청) | $3.83 | $4.47 | +$0.64 (+17%) | 156 → 134건 |
| 50% | $3.93 | $4.76 | +$0.83 (+21%) | 153 → 126건 |
| 100% (항상 5분 넘게 쉬었다 요청) | $4.03 | $5.05 | +$1.02 (+25%) | 149 → 119건 |

캐시 쓰기($5)가 일반 입력($4)보다도 비싸므로, **사용자가 쉬었다가 요청하는 습관이 있을수록 MCP 고정 영역의 비용이 커진다.** 추가된 80K는 만료될 때마다 80K × $5/M = $0.40을 더 쓴다.

### 4.4 작업 중 5분 넘는 도구 실행

빌드나 테스트처럼 5분 넘게 걸리는 도구가 작업 중 한 번 끼면(만료 비율 50% 기준):

| | A | B | 차이 |
|---|---|---|---|
| 긴 도구 실행 없음 | $3.93 | $4.76 | +$0.83 |
| 긴 도구 실행 1회 | $4.15 | $5.37 | **+$1.22** |

만료 한 번마다 차이가 약 $0.40씩 더 벌어진다. 만료 시점에는 그때까지 쌓인 대화도 함께 다시 써야 하므로, 실제로는 세션 후반에 만료될수록 절대 비용이 더 크다.

### 4.5 루프 길이(호출 수)의 영향

| 작업당 호출 수 | A | B | 차이 |
|---|---|---|---|
| 20회 | $3.25 | $3.76 | +$0.51 (+16%) |
| 40회 | $3.93 | $4.76 | +$0.83 (+21%) |
| 80회 | $5.29 | $6.76 | +$1.47 (+28%) |

```
작업당 차이 ≈ 추가 컨텍스트 × 캐시 읽기 단가 × 호출 수  +  추가 컨텍스트 × 캐시 쓰기 단가 × 만료 횟수
            = 80K × $0.20/M × N                       +  80K × $5.00/M × (만료 횟수)
            = $0.016 × N                              +  $0.40 × (만료 횟수)
```

- **호출 수**(에이전트가 도구를 많이 쓰는 작업일수록)가 늘면 캐시 읽기 쪽 차이가 커진다.
- **만료 횟수**(사용자 대기, 긴 빌드)가 늘면 캐시 쓰기 쪽 차이가 커진다.
- 코딩 에이전트는 두 가지가 모두 흔하다.

### 4.6 GitHub Copilot 크레딧으로 환산

2026년 6월 1일부터 Copilot은 premium request 대신 토큰 기반 GitHub AI Credits(1 credit = $0.01)로 과금한다.

| | A | B |
|---|---|---|
| 작업 1건당 | 약 393 credits | 약 476 credits |
| 차이 | | **약 +83 credits** |

Copilot Business 요금제 기본 크레딧($19/월)을 기준으로 보면 Opus 에이전트 작업은 몇 건에 불과하므로, 고정 컨텍스트 관리가 팀 예산에 직접 영향을 준다. (Copilot에서 Opus 5.5에 적용되는 실제 단가는 GitHub의 모델 가격표에서 확인 필요)

---

## 5. 해결책 1: MCP 서버를 컨텍스트 효율적으로 설계하기 (3계층)

### 5.1 MCP best practice가 말하는 원칙

MCP 문서의 [모범 사례](https://modelcontextprotocol.info/ko/docs/best-practices/)와 [효과적인 도구 작성 가이드](https://modelcontextprotocol.info/ko/docs/tutorials/writing-effective-tools/)는 다음 원칙을 제시한다.

| 원칙 | 출처 문서 | 컨텍스트 관점의 의미 |
|---|---|---|
| 단일 책임 원칙(SRP): 서버마다 명확한 목적 하나 | 모범 사례 | 거대한 "메가 서버" 대신 목적별로 나누면 필요한 서버만 쓸 수 있다 |
| API 엔드포인트마다 도구를 만들지 말 것 | 효과적인 도구 작성 | `list_users`, `list_events`, `create_event` 대신 `schedule_event` 하나처럼 워크플로 단위로 통합 → 도구 수 감소 |
| 도구가 많다고 좋은 것이 아님 | 효과적인 도구 작성 | 고임팩트 워크플로에 맞춘 소수의 도구부터 시작 |
| 네임스페이싱 | 효과적인 도구 작성 | 공통 접두사로 관련 도구를 묶어 경계를 분명히 → 검색과 선택이 쉬워짐 |
| 토큰 효율적인 응답 | 효과적인 도구 작성 | 응답의 양을 최적화, concise/detailed 응답 형식 제공 |

> 참고: 이 문서들이 "3계층"이라는 용어를 직접 쓰지는 않는다. 아래 3계층 패턴은 위 원칙(도구 수 최소화, 워크플로 단위 통합, 토큰 효율)을 **도구 정의가 차지하는 컨텍스트**에까지 적용한 설계 방식이다.

### 5.2 3계층 패턴

도구 정의를 미리 다 노출하지 않고, 필요할 때 단계적으로 공개(progressive disclosure)한다.

```
1층  발견:   list_categories()          → "jira", "confluence", "bitbucket" ...
2층  조회:   describe_tool("jira.search") → 그 작업의 입력 스키마만 반환
3층  실행:   execute_tool("jira.search", {...})
```

```python
# 서버가 모델에 노출하는 도구는 3개뿐
@mcp.tool()
def list_categories() -> list[str]:
    """Return available capability categories (e.g. jira, confluence)."""
    return list(REGISTRY.keys())

@mcp.tool()
def describe_tool(name: str) -> dict:
    """Return the input schema and usage notes for one operation."""
    return REGISTRY_SCHEMAS[name]

@mcp.tool()
def execute_tool(name: str, args: dict) -> dict:
    """Run an operation by name with arguments matching describe_tool's schema."""
    validate(args, REGISTRY_SCHEMAS[name])   # 서버 쪽에서 반드시 검증
    return REGISTRY[name](**args)
```

- 대기 상태의 컨텍스트: 도구 수십 개(수만 토큰) → 메타 도구 3개(수백 토큰)
- 대가: 발견과 조회에 왕복이 1\~2회 추가되고, `args: dict`라서 클라이언트 쪽 스키마 검증과 도구 단위 권한 제어가 약해진다.

### 5.3 현실적인 한계

대부분의 MCP 서버(GitHub, Jira, Slack, Context7 등)는 **제공받아 쓰는 것**이다. 사용자가 그 서버들을 3계층으로 바꿀 수는 없다. 직접 만드는 사내 MCP에는 적용할 수 있지만, 이미 쓰고 있는 외부 MCP의 고정 비용은 그대로 남는다.

그래서 **클라이언트(코딩 에이전트) 쪽에서 같은 효과를 내는 기능**이 필요하다.

---

## 6. 해결책 2: 코딩 에이전트의 defer loading(tool search)

### 6.1 동작 원리

3계층 패턴이 서버 안에서 하던 일을 클라이언트와 API가 대신 한다.

```
[세션 시작]
  컨텍스트: built-in 도구 정의 + MCP 도구 "이름"과 서버 instructions만   ← 수 K 토큰
[작업 중 Jira가 필요해짐]
  모델 → ToolSearch("jira issue search")
       → 이름/description/인자 설명으로 검색, 상위 몇 개 반환
       → 찾은 도구의 전체 정의를 대화에 추가
  모델 → mcp__jira__search_issues 호출
```

| 단계 | 모델이 보는 것 |
|---|---|
| ① 검색할지 결정 | 도구 **이름**과 **서버 instructions**만. 이름이 모호하면 검색 자체를 안 할 수 있다 |
| ② 검색 | 이름, description, 인자 이름과 인자 설명을 대상으로 매칭. 기본 최대 5개 반환 |
| ③ 호출 | 찾은 도구의 전체 스키마를 보고 호출 |

중요한 특성:

- **로딩된 도구는 이후 턴에서도 유지된다.** 다시 검색할 필요 없이 계속 쓸 수 있고, 그동안 input 토큰으로 계속 계산된다.
- **compaction이 일어나면 풀린다.** 다시 필요하면 다시 검색한다.
- 찾은 정의는 시스템 prefix가 아니라 **대화 중간에 추가**된다(`tool_reference` 블록을 API가 전체 정의로 확장). prefix가 바뀌지 않으므로 **기존 캐시가 깨지지 않는다.**

### 6.2 효과 (같은 가정으로 계산)

선택 MCP 8%(80K)를 defer로 돌려 이름만 약 3K가 실리고, 작업 중간(20번째 호출)에 도구 2개(4K)를 불러오는 경우:

| 구성 | 작업당 비용 |
|---|---|
| A: 필수만 (4%) | $3.93 |
| B: 필수 + 선택 MCP, defer 없음 (12%) | $4.76 |
| B + tool search, 선택 MCP를 이번 작업에서 안 씀 | **$3.96** |
| B + tool search, 선택 MCP 도구 2개를 씀 | **$4.00** |

```python
def ts_cost(cold, use_tool):
    base = 40_000 + 3_000                    # 필수 + 선택 MCP의 도구 "이름"만
    step = GROWTH / CALLS
    reads = writes = 0.0
    for i in range(1, CALLS + 1):
        extra = 4_000 if (use_tool and i > 20) else 0   # 20번째 호출 이후 로딩된 도구 정의
        prefix = base + (i - 1) * step + extra
        if i == 1:
            if cold: writes += base
            else:    reads  += base
        else:
            reads += prefix
    writes += GROWTH + (4_000 if use_tool else 0)       # 로딩된 정의는 한 번 캐시 쓰기
    return (reads * P_READ + writes * P_WRITE + OUT * P_OUT) / 1e6
```

**선택 MCP를 연결해 둔 채로도 비용이 필수만 쓸 때와 거의 같아진다.** 대신 도구가 필요할 때마다 검색 왕복이 한 번 추가된다.

---

## 7. Claude Code 설정 상세

### 7.1 전체 동작: `ENABLE_TOOL_SEARCH`

| 값 | 동작 |
|---|---|
| (설정 안 함) | 모든 MCP 도구를 defer. **단, `ANTHROPIC_BASE_URL`이 Anthropic이 아닌 호스트면 꺼짐**(전부 미리 로딩) |
| `true` | 강제로 defer. 프록시에도 beta header를 보내며, 프록시가 `tool_reference`를 지원하지 않으면 요청 실패 |
| `auto` | defer 대상 도구 정의 합계가 컨텍스트의 10% 이상이면 defer, 미만이면 미리 로딩 |
| `auto:N` | 기준을 N%로 지정 (예: `auto:5`) |
| `false` | 끔. 모든 도구 정의를 매 턴 실음 |

```bash
# 일회성
ENABLE_TOOL_SEARCH=true claude
```

```json
// ~/.claude/settings.json (사용자 전체) 또는 .claude/settings.json (프로젝트)
{
  "env": {
    "ENABLE_TOOL_SEARCH": "true"
  }
}
```

기타:

- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`가 설정되어 있으면 tool search는 꺼지고 `ENABLE_TOOL_SEARCH`로 되돌릴 수 없다.
- 조직 전체에 강제하려면 managed settings를 사용한다 (v2.1.227 이상).
- `ToolSearch` 도구 자체를 막을 수도 있다: `{"permissions": {"deny": ["ToolSearch"]}}`
- 지원 모델: Claude Opus 4.5 / Sonnet 4.5 / Haiku 4.5 이후 모델 (Opus 5.5 포함)

### 7.2 사내 환경(base URL = GitHub Copilot API)

base URL을 `https://api.githubcopilot.com`으로 바꿔 쓰면, 설정하지 않은 상태에서는 **tool search가 꺼져 모든 MCP 정의가 매 호출 실린다.** (Claude Code MCP 문서 "Configure tool search", "Tool availability" 절에 명시)

사내 검증 결과, `ENABLE_TOOL_SEARCH=true`로 강제하면 Copilot API 경유에서도 MCP 도구가 on-demand 상태로 바뀌고, 필요한 도구만 로딩되어 정상 사용되는 것을 확인했다. 권장 설정:

```json
{
  "apiKeyHelper": "gh auth token",
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.githubcopilot.com",
    "ENABLE_TOOL_SEARCH": "true"
  }
}
```

추가로 확인해 둘 항목:

| 항목 | 확인 방법 |
|---|---|
| compaction 후 재검색 | 긴 세션에서 `/compact` 후 같은 MCP 도구가 필요한 요청 |
| 서브에이전트 | 서브에이전트에게 MCP 도구가 필요한 작업을 맡김 |
| `alwaysLoad` 혼용 | 핵심 서버 하나만 `alwaysLoad: true`로 두고 `/context` 확인 |
| 검색 누락 | 이름이 모호한 MCP 도구가 필요한 요청을 반복해 봄 |

### 7.3 서버별 예외: `alwaysLoad` (사용자 설정)

```json
// .mcp.json
{
  "mcpServers": {
    "core-tools": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "alwaysLoad": true
    },
    "nice-to-have": {
      "command": "npx",
      "args": ["-y", "@foo/mcp"],
      "alwaysLoad": false
    }
  }
}
```

| 값 | 동작 |
|---|---|
| `true` | 이 서버의 도구는 `ENABLE_TOOL_SEARCH`와 상관없이 항상 미리 로딩. 시작 시 이 서버 연결을 최대 5초 기다림 |
| `false` | 서버 작성자가 일부 도구를 미리 로딩하도록 지정했어도 전부 defer (v2.1.287 이상) |
| 생략 | 기본 동작. 작성자가 `anthropic/alwaysLoad`로 표시한 도구만 미리 로딩 |

매 턴 쓰는 소수의 서버만 `true`로 둔다. 미리 로딩한 도구는 그만큼 고정 비용이 된다.

### 7.4 도구별 지정: `anthropic/alwaysLoad` (서버 작성자 설정)

직접 MCP 서버를 만든다면, 핵심 도구만 미리 로딩되도록 `tools/list` 응답에 표시할 수 있다.

```json
{
  "name": "search_tickets",
  "description": "Searches the ticket tracker by keyword",
  "_meta": { "anthropic/alwaysLoad": true }
}
```

### 7.5 검색에 잘 걸리게 하기

defer 상태에서 모델은 ①단계에서 **이름과 서버 instructions만** 보고 검색 여부를 정한다.

- 이름: `query_slack`보다 `search_slack_messages`처럼 동작과 대상이 드러나게
- description: 사용자가 실제로 쓸 키워드를 앞쪽에 (도구 description과 서버 instructions는 기본 2,048자에서 잘림. `CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH`로 조정, v2.1.280 이상)
- 서버 instructions: "어떤 작업일 때 이 서버의 도구를 검색해야 하는지"를 명시
- 시스템 프롬프트나 CLAUDE.md에 도구 카테고리 안내: "Slack, Jira, Confluence용 도구를 검색해서 쓸 수 있다"

### 7.6 Agent SDK에서 쓰는 경우

```python
options = ClaudeAgentOptions(
    setting_sources=["project"],              # 프로젝트 루트의 .mcp.json 로딩
    env={"ENABLE_TOOL_SEARCH": "auto:5"},     # 정의가 컨텍스트 5% 넘으면 defer
)
```

`settingSources`를 직접 지정할 때는 `"project"`를 포함해야 `.mcp.json`이 읽힌다.

### 7.7 확인 방법

```bash
claude
> /context    # "MCP tools" 항목 크기 확인: defer 상태면 이름 수준으로 작아짐
> /mcp        # 서버별 연결 상태, 도구 수 확인
```

---

## 8. 다른 코딩 에이전트의 유사 옵션

### 8.1 GitHub Copilot CLI

- tool search가 **기본으로 켜져 있다.** 프롬프트마다 도구 수를 확인해, 대략 30개 미만이면 전부 로딩하고 그 이상이면 built-in 도구만 미리 싣고 MCP 등 외부 도구는 보류한다.
- 한 번 불러온 도구는 **대화가 끝날 때까지 유지**된다.
- 끄기: 개인 설정(`~/.copilot/settings.json`)에 `"toolSearch": false`
- 서버별 예외: `~/.copilot/mcp-config.json`의 `deferTools` (`"auto"` 기본값, `"never"`는 항상 로딩). `/mcp edit`에서도 변경 가능

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@example/mcp-server"],
      "deferTools": "never",
      "tools": ["*"]
    }
  }
}
```

- 커스텀 에이전트(`.github/agents/*.md`)가 도구를 이름으로 나열한 경우, frontmatter에 `deferred-tool-loading: true`를 넣으면 그 목록도 검색 대상으로 돌린다.
- 지원 모델: Claude Haiku 4.5+, Sonnet/Opus 4.0+, Fable, GPT-5.4 이후 등

### 8.2 OpenAI Codex CLI

- Codex에도 BM25 기반 `tool_search`가 내장되어 있어, MCP 도구를 바로 노출할지(direct) 검색 뒤로 미룰지(deferred) 나누어 다룬다.
- defer된 MCP 도구가 검색될 때 서버 instructions가 네임스페이스 설명으로 쓰이므로, Claude Code와 마찬가지로 **서버 instructions의 품질**이 검색 성공률에 영향을 준다.
- compaction 시 검색 결과가 버려지므로 이후에는 다시 검색한다.
- MCP 서버 설정은 `~/.codex/config.toml`의 `[mcp_servers.<이름>]` 섹션에서 한다. defer 동작 관련 세부 옵션 키는 버전에 따라 바뀌고 있으므로 [Codex configuration reference](https://developers.openai.com/codex/config-reference)에서 사용 중인 버전 기준으로 확인한다.

---

## 9. 정리: 실천 체크리스트

1. `/context`로 현재 세션의 MCP tools 크기를 확인한다.
2. Claude Code에 `ENABLE_TOOL_SEARCH=true`를 `settings.json`에 고정한다. (사내 Copilot base URL 환경에서도 동작 확인됨)
3. 매 작업에 반드시 쓰는 서버만 `alwaysLoad: true`, 나머지는 defer로 둔다.
4. 정말 안 쓰는 MCP는 `/mcp`에서 비활성화한다. defer여도 이름과 instructions는 실린다.
5. 직접 만드는 MCP는 best practice(SRP, 워크플로 단위 도구, 네임스페이싱, 토큰 효율 응답)를 따르고, 이름·description·서버 instructions를 검색되기 좋게 쓴다.
6. 캐시 관점의 사용 습관:
   - 같은 맥락의 후속 요청은 5분 안에 이어서 보낸다.
   - 오래 쉬었다가 다른 주제를 시작할 거라면 `/clear`나 새 세션으로 시작해 쌓인 대화를 다시 쓰지 않게 한다.
   - 5분 넘는 빌드·테스트는 백그라운드로 돌리거나, 결과만 짧게 요약해 돌려주도록 한다.

---

## 출처

- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp) — tool search 설정, `ENABLE_TOOL_SEARCH`, `alwaysLoad`, base URL 조건
- [Connect to external tools with MCP (Agent SDK)](https://code.claude.com/docs/en/agent-sdk/mcp) — `.mcp.json`과 `settingSources`
- [Scale to many tools with tool search (Agent SDK)](https://code.claude.com/docs/en/agent-sdk/tool-search) — 로딩 유지와 compaction, 50개 도구 10\~20K 토큰
- [Tool search tool (Claude API)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) — 검색 대상 필드, `tool_reference`, 캐시 보존, 55K 예시
- [MCP 모범 사례: 아키텍처 & 구현 가이드](https://modelcontextprotocol.info/ko/docs/best-practices/)
- [에이전트를 위한 효과적인 도구 작성: MCP 개발 완전 가이드](https://modelcontextprotocol.info/ko/docs/tutorials/writing-effective-tools/)
- [Loading tools on demand with tool search (GitHub Copilot CLI)](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/tool-search)
- [GitHub Copilot is moving to usage-based billing (GitHub Blog)](https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/)
- [Use MCP server instructions in deferred namespace descriptions (openai/codex #21053)](https://github.com/openai/codex/issues/21053)
- Opus 5.5 단가(입력 $4, 캐시 쓰기 $5, 캐시 읽기 $0.20, 출력 $20 / 1M)는 사용자 제공 값 기준
