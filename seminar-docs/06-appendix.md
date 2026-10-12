# 부록

> 범위: 부록 A~E · 기준 시점: 2026년 10월

부록은 본문의 흐름을 끊지만 실제로 손을 움직일 때 필요한 것들을 모아 둔 곳이다.

| 부록 | 내용 | 언제 쓰나 |
| --- | --- | --- |
| A. 설치와 백엔드 연결 | 설치, 사내 백엔드 연결, tool search 설정, 트러블슈팅 | 첫 셋업 때 (7.6절의 상세) |
| B. 핸즈온 랩 | 12개 실습. 앞 랩의 결과 위에 쌓인다 | 실습 시간에 |
| C. 심화 읽기 | 본문에서 덜어 낸 다섯 주제와 비용 계산 모델 | 발표 뒤에 |
| D. 용어집 | 이 세미나가 쓰는 용어의 정의 | 읽는 중에 |
| E. 참고 자료 | 장 번호 순으로 모은 출처 | 확인할 때 |

주의 한 줄. **부록 A가 이 문서에서 가장 먼저 낡는다.** 설정 키와 버전은 몇 달 사이에 바뀐다. 3.2절에서 말한 대로 값을 외우지 말고 확인하는 방법을 가져간다. 아래의 모든 설정값은 2026년 10월에 공식 문서로 확인한 것이다.

---

## 부록 A. 설치와 백엔드 연결

### A.1 버전과 전제

본문이 설명한 동작은 아래 버전을 전제로 한다. 사내 환경은 LLM 게이트웨이형 연결이므로 더 높은 버전이 필요한 항목이 있다.

| 기능 | 필요 버전 | 본문 |
| --- | --- | --- |
| `AGENTS.md` 직접 읽기 | v2.1.277 이상 | 3.4절, 5.2절 |
| 일부 세션(Amazon Bedrock, 텔레메트리 끈 세션 등)에서도 `AGENTS.md` 직접 읽기 | v2.1.281 이상 | 3.4절 |
| `/context`에 직접 읽은 `AGENTS.md` 표시 | v2.1.280 이상 | 1.5절 |
| `auto` 시작 모드 (대화형 터미널·VS Code) | v2.1.283 이상 | 7.5절 |
| 서버별 `alwaysLoad: false` | v2.1.287 이상 | 4.8절 |
| 훅의 `onFailure: "block"` | v2.1.295 이상 | 5.5절 |
| 조직 차원의 tool search 강제(managed settings) | v2.1.227 이상 | 4.8절 |

**기준: v2.1.283 이상.** 훅의 실패 차단까지 쓰려면 v2.1.295 이상으로 올린다. 설치와 업데이트는 각자 환경의 승인된 배포 경로를 따른다.

```bash
claude --version      # 버전 확인
gh auth status        # Copilot 자격증명 확인
```

### A.2 Claude Code + Copilot 백엔드 (사내 공유 방식)

사용자 설정 파일 `~/.claude/settings.json`에 세 가지를 넣는다. 연결 주소, 토큰을 가져오는 방법, 그리고 tool search다.

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.githubcopilot.com",
    "ENABLE_TOOL_SEARCH": "true"
  },
  "apiKeyHelper": "gh auth token"
}
```

- **`ANTHROPIC_BASE_URL`**: Claude Code가 요청을 보낼 주소다. 사내에서는 Copilot API를 가리킨다. 이 주소는 공식 문서에 나오는 예가 아니라 사내에서 동작을 확인한 구성이다.
- **`apiKeyHelper`**: 매 요청 때 실행해 토큰을 받아 오는 명령이다. 토큰을 설정 파일에 평문으로 적지 않으므로 만료와 회수, 감사가 정상 동작한다.
- **`ENABLE_TOOL_SEARCH`**: 기본값으로는 꺼지는 환경이어서 직접 켠다(A.3).

연결 확인은 세 단계다.

```bash
claude --version
claude -p "print hello world in C"                 # 비대화형 1회 요청
claude -p "hello" --output-format json | tail -20  # usage 필드 확인 (캐시 적중 여부)
```

세 번째 명령을 두 번 이어서 실행하고, 두 번째 결과의 `usage`에서 `cache_read_input_tokens`가 0이 아닌지 본다. 0이면 게이트웨이가 캐시 표식을 전달하지 않는 것이고, 4.5절과 4.8절의 계산값은 이 환경에 맞지 않는다(그때는 고정 영역을 줄이는 효과가 계산보다 커진다).

> **발표 전 확인**: 게이트웨이를 거치는 세션에서는 기능이 일부 빠질 수 있다. ① `/config`에 Project instructions 항목이 보이는지(없으면 `AGENTS.md` 직접 읽기가 불가해 5.3절 끝의 `@AGENTS.md` 방식을 쓴다), ② prompt caching이 실제로 적중하는지, ③ auto 모드 분류기가 동작하는지를 실제 세션으로 확인한다.

### A.3 tool search 설정

4.8절의 설정을 한곳에 모았다. 값은 `settings.json`의 `env`에 두거나 한 번만 쓰려면 환경 변수로 준다.

```bash
ENABLE_TOOL_SEARCH=true claude      # 일회성
ENABLE_TOOL_SEARCH=auto:5 claude    # 임계값 5%
```

| 값 | 동작 |
| --- | --- |
| (설정 안 함) | 모든 MCP 도구를 미룬다. 단 `ANTHROPIC_BASE_URL`이 자사 것이 아닌 호스트이거나, Microsoft Foundry(Azure) 배포, Google Cloud Agent Platform의 Claude 4.5 이전 세대 모델이면 전부 미리 싣는다 |
| `true` | 모든 MCP 도구를 미룬다. 프록시에도 베타 헤더를 보내므로, 프록시가 `tool_reference` 블록을 지원하지 않으면 요청이 실패한다 |
| `auto` | 미룰 수 있는 정의의 합이 컨텍스트 윈도우의 10% 미만이면 미리 싣고, 10%에 이르면 전부 미룬다 |
| `auto:N` | 임계값을 N%(0~100)로 정한다 |
| `false` | 전부 미리 싣는다 |

(출처: [Claude Code 문서, Configure tool search](https://code.claude.com/docs/en/mcp#configure-tool-search), 2026-10 확인)

함께 알아 둘 것.

- **모델 요건**: `tool_reference` 블록을 지원하는 모델이어야 한다. Claude Sonnet 4.5, Haiku 4.5, Opus 4.5와 그 이후 모델이다.
- **`CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`** 가 설정돼 있으면 tool search는 꺼진 채로 남고 `ENABLE_TOOL_SEARCH`로 덮어쓸 수 없다. 조직은 managed settings로 켜 둘 수 있다(v2.1.227 이상).
- **`ToolSearch` 도구 자체를 막을 수도 있다**: `{"permissions": {"deny": ["ToolSearch"]}}`.

매 턴 쓰는 서버만 미루기에서 뺀다. `.mcp.json`의 그 서버 항목에 `alwaysLoad: true`를 둔다. 미리 실은 만큼 고정 비용이 되므로 소수에만 쓴다.

```json
{
  "mcpServers": {
    "core-tools": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "alwaysLoad": true
    },
    "nice-to-have": {
      "command": "npx",
      "args": ["-y", "@example/mcp-server"],
      "alwaysLoad": false
    }
  }
}
```

`alwaysLoad: true`인 서버는 첫 프롬프트를 만들 때 도구가 있어야 하므로 시작할 때 최대 5초까지 연결을 기다린다. `alwaysLoad: false`는 서버 작성자가 미리 로드하도록 표시한 도구까지 모두 미루게 한다(v2.1.287 이상).

확인과 점검.

```text
/context      # MCP tools 행이 이름 수준으로 작아졌는지
/mcp          # 서버별 연결 상태와 도구 수
```

| 확인 항목 | 방법 |
| --- | --- |
| 실제 호출이 되는가 | MCP 도구가 필요한 요청을 한 번 보내 오류 없이 끝나는지 본다 |
| 압축 뒤 재검색 | 긴 세션에서 `/compact` 뒤에 같은 MCP 도구가 필요한 요청을 해 본다 |
| 서브에이전트 | 서브에이전트에게 MCP 도구가 필요한 작업을 맡겨 본다 |
| 검색 누락 | 이름이 모호한 도구가 필요한 요청을 반복해 본다 |

직접 MCP 서버를 만든다면 검색에 걸리게 쓰는 것이 중요하다. 도구 이름은 `query_slack`보다 `search_slack_messages`처럼 동작과 대상이 드러나게 쓰고, 설명과 서버 instructions는 앞쪽에 핵심 용례를 둔다. 둘은 기본 2,048자에서 잘리고, 한도는 `CLAUDE_CODE_MAX_MCP_DESCRIPTION_LENGTH`로 바꾼다(v2.1.280 이상). 서버 작성자는 핵심 도구만 미리 로드되게 `tools/list` 응답의 `_meta`에 `"anthropic/alwaysLoad": true`를 표시할 수 있다.

### A.4 Codex CLI + 커스텀 프로바이더

가장 흔한 함정부터 적는다. **`wire_api`의 유효값은 `responses` 하나뿐이고, 생략하면 그것이 기본값이다.** 2025년에 쓰인 가이드에 흔히 남아 있는 `wire_api = "chat"`은 지금 설정에서 지워야 한다(출처: [Codex 설정 참조](https://learn.chatgpt.com/docs/config-file/config-reference), 2026-10 확인).

```toml
# ~/.codex/config.toml  (반드시 사용자 레벨)
model          = "<model-id>"
model_provider = "gateway"

[model_providers.gateway]
name     = "Internal Gateway"
base_url = "https://<responses-compatible-endpoint>/v1"
env_key  = "GATEWAY_API_KEY"
# wire_api 는 생략한다 (기본값 responses)
```

- **핵심 제약**: `base_url`이 가리키는 엔드포인트가 Responses API를 말해야 한다. Chat Completions만 제공하는 엔드포인트라면 변환 게이트웨이를 사이에 둬야 한다.
- **토큰을 파일에 적지 않으려면** `env_key` 대신 명령으로 토큰을 받는 `auth` 블록을 쓴다. Claude Code의 `apiKeyHelper`와 같은 역할이고, `env_key`와 함께 쓰지 않는다.
- **설정 우선순위**(위가 이긴다): CLI 플래그 → 프로필 → 프로젝트 `.codex/config.toml`(신뢰된 프로젝트에서만) → 사용자 `~/.codex/config.toml`. **프로바이더와 인증 블록은 반드시 사용자 레벨에 둔다.** 프로젝트 레벨에 두면 무시되고 경고만 뜬다. 프로젝트별 지시는 설정이 아니라 `AGENTS.md`에 쓴다.

> **발표 전 확인**: 사내 Copilot API가 Responses 라인을 노출하는지는 조직 설정에 따라 다를 수 있다. Claude Code 쪽(A.2)은 Anthropic 호환 라인으로 바로 붙지만, Codex는 Responses 호환 여부를 먼저 확인하고 아니면 변환 게이트웨이를 두는 것이 전제다. 발표 전에 실제로 붙여 보고 결과를 확정해 넣는다.

### A.5 트러블슈팅 카탈로그

| 증상 | 원인 후보 | 확인 |
| --- | --- | --- |
| 401 | 토큰 만료 또는 권한 | `gh auth status`. `apiKeyHelper`가 토큰을 돌려주는지 직접 실행해 본다 |
| 404, unknown endpoint, 빈 스트림 | 엔드포인트가 기대한 프로토콜을 말하지 않는다 | 같은 호스트에 다른 라인은 되는지 본다. Codex는 Responses 호환 여부(A.4) |
| MCP 도구를 쓰는 요청만 실패 | 프록시가 `tool_reference` 블록을 전달하지 않는다 | `ENABLE_TOOL_SEARCH`를 빼고 다시 시작해 본다(4.8절) |
| 모델 없음 | 모델 ID 오타, 조직 정책상 미허용 | 사용 가능 모델 목록을 조회한다 |
| 인증서 오류 | 사내 프록시의 TLS 검사 | CA 번들을 등록한다 |
| 캐시가 적중하지 않는다 | 게이트웨이가 `cache_control`을 지운다 | `--output-format json`의 `usage` 또는 `/usage`의 `Prompt cache (main)` 행(v2.1.251 이상) |
| `AGENTS.md`를 읽지 않는다 | `CLAUDE.md`나 `CLAUDE.local.md`가 있다. 버전이 낮다. 게이트웨이 세션 | 시작 줄과 `/context`의 `Memory files` 확인(3.4절) |
| 지침이 지켜지지 않는다 | 로드되지 않았거나 압축으로 빠졌다 | `/context`로 로드 여부, 압축 발생 여부 확인(1.3절) |
| 훅이 동작하지 않는다 | 경로 오타나 스크립트 오류. 실패해도 통과된다 | 디버그 로그를 보고, 정책용 훅에는 `onFailure: "block"`을 건다(5.5절) |
| TUI가 깨진다 | 터미널 호환성 | `TERM` 확인, 다른 터미널로 재시도 |

### A.6 첫 세션 점검

7.6절의 일곱 단계에 확인 명령을 붙인 것이다. 위에서부터 한다.

| 순서 | 할 일 | 확인 |
| --- | --- | --- |
| 1 | 도구를 고른다 | 고른 이유를 네 축으로 한 줄로 말할 수 있다(3.2절) |
| 2 | 백엔드에 연결한다 | `claude --version`, `claude -p "..."` (A.2) |
| 3 | 권한을 설정한다 | `/status`로 시작 모드와 규칙을 본다. 되돌릴 수 없는 명령이 `ask` 또는 `deny`에 있다 |
| 4 | 지침 파일을 만든다 | `AGENTS.md` 하나, 200줄 아래 |
| 5 | `/context`로 확인한다 | `Memory files`에 지침이 보이고, 가장 큰 행을 안다 |
| 6 | tool search를 설정한다 | 설정 전후의 `MCP tools` 행을 비교했다 (A.3) |
| 7 | 첫 Hook을 건다 | 일부러 틀린 코드를 쓰게 해서 검사가 걸리는 것을 봤다 |

---

## 부록 B. 핸즈온 랩

사내 리포지토리의 축소판으로 진행한다. 각 랩은 앞 랩의 결과 위에 쌓인다. 1~5번만 해도 혼자서 쓸 수 있는 환경이 된다.

| # | 주제 | 산출물 | 관련 절 |
| --- | --- | --- | --- |
| 1 | 환경 구성 | 백엔드에 연결된 첫 세션 | 부록 A, 7.6 |
| 2 | 지침 작성 | 리포지토리 루트의 `AGENTS.md` | 5.2 |
| 3 | 지연 로드 | `paths` 조건부 룰 2개 분리 | 5.3 |
| 4 | 첫 위임 | Plan mode로 실패 테스트 1건 수정 | 7.5 |
| 5 | **실패 복구** | 잘못된 결과를 되돌리고 다시 지시 | 7.5, 10.5 |
| 6 | 요청 인터뷰 | 인터뷰로 만든 `SPEC.md` | 12.5 |
| 7 | 스킬 작성 | 코드 리뷰 체크리스트 스킬 | 5.4 |
| 8 | 훅 작성 | 편집 후 포맷과 정적 분석 | 5.5 |
| 9 | 서브에이전트 | 읽기 전용 검토자 정의와 실행 | 6.3 |
| 10 | 비용 측정 | `/context`와 크레딧 대조 | 1.5, 4.8, 4.11 |
| 11 | 패키징 | 위 자산을 플러그인으로 묶기 | 6.4 |
| 12 | (선택) 첫 루프 | 검증 가능한 종료 조건 하나 돌려보기 | 부록 C.5 |

**랩 5가 가장 중요하다.** 에이전트가 틀렸을 때 어떻게 하는지를 몸에 익히지 않으면 나머지가 쓸모없다.

### 랩 1. 환경 구성

- **목표**: 승인된 백엔드에 붙여 첫 응답을 받는다.
- **하는 일**: 부록 A.2의 설정을 넣고 `claude -p`로 1회 요청을 보낸다. `/context`를 실행해 아무 요청도 하지 않은 상태의 고정 영역을 적어 둔다.
- **완료 조건**: 응답이 오고, `/context`의 가장 큰 행을 말할 수 있다.

### 랩 2. 지침 작성

- **목표**: 모델이 추측할 수 없는 것만 적는다.
- **하는 일**: 5.2절의 예시를 틀로 삼아 `AGENTS.md`를 쓴다. 빌드·테스트 명령, 이 코드베이스의 함정 세 줄, 판단 기준 세 줄로 시작한다. 줄마다 "이 줄을 지우면 에이전트가 실수하게 되는가"를 묻는다.
- **이미 지침이 있는 저장소라면**: 새로 쓰기 전에 `/doctor prompt-audit`을 돌려 보고서를 읽는다. 낡은 줄과 모순되는 줄이 어디에 있었는지 확인하고, 한 묶음만 지운 뒤 같은 작업을 다시 시켜 본다(5.2절).
- **완료 조건**: 200줄 아래이고, `/context`의 `Memory files`에 잡힌다. 지침에 적은 함정과 관련된 작업을 시켜 그 규칙이 지켜지는지 본다.

### 랩 3. 지연 로드

- **목표**: 항상 필요하지는 않은 내용을 아래 칸으로 내린다.
- **하는 일**: 랩 2의 지침에서 특정 경로에만 해당하는 규칙 두 덩어리를 골라 `.claude/rules/` 아래 파일로 옮기고 `paths:`를 지정한다. 해당 경로의 파일을 다룰 때 로드되는지 확인한다.
- **완료 조건**: 루트 지침이 줄었고, 해당 경로를 건드리지 않는 세션에서는 그 규칙이 로드되지 않는다. `@`를 붙이면 시작 때 전부 로드된다는 것도 한 번 확인한다(5.3절).

### 랩 4. 첫 위임

- **목표**: 계획을 받고 승인한 뒤 맡기는 순서를 익힌다.
- **하는 일**: 실패하는 테스트 하나를 고르고, Plan mode로 계획을 받는다. 계획을 한 군데 고친 뒤 승인하고 구현을 맡긴다.
- **완료 조건**: 테스트가 통과하고, 계획과 결과의 차이를 설명할 수 있다.

### 랩 5. 실패 복구

- **목표**: 틀린 결과를 싸게 버리는 법을 익힌다.
- **하는 일**: 일부러 모호한 요청을 보내 엉뚱한 결과를 만든다. `Esc` 두 번 또는 `/rewind`로 되돌리고, 셸 명령으로 바뀐 것은 되돌아오지 않는다는 것을 확인한다. 그다음 `/clear`로 컨텍스트를 비우고, 배운 것을 담아 요청을 다시 쓴다.
- **완료 조건**: 되돌리기와 새 세션의 차이를 말할 수 있고, 다시 쓴 요청이 한 번에 통과한다.

### 랩 6. 요청 인터뷰

- **목표**: 요청에서 빠진 것을 구현 전에 드러낸다.
- **하는 일**: 12.5절의 시연 순서를 그대로 따라 한다. 인터뷰 스킬이나 대체 프롬프트로 한 줄 요청을 인터뷰하고, 합의한 내용을 `SPEC.md`로 남긴다. 추천 답을 뒤집는 답을 적어도 하나 넣는다.
- **완료 조건**: `SPEC.md`에 완료 기준과 범위 밖 항목이 있고, 처음 한 줄 요청에 없던 제약이 세 개 이상 적혀 있다.

### 랩 7. 스킬 작성

- **목표**: 평소에는 한 줄, 쓸 때만 본문이 실리는 자산을 만든다.
- **하는 일**: 팀의 코드 리뷰 체크리스트를 `SKILL.md`로 만든다. `description`에 "언제 써야 하는지"를 쓰고, 긴 참고 자료는 별도 파일에 두고 일반 링크로 가리킨다.
- **완료 조건**: 관련된 요청에서 스킬이 저절로 불려 오고, 부르지 않은 세션의 `/context`에서는 설명 한 줄만 보인다.

### 랩 8. 훅 작성

- **목표**: 요청을 강제로 바꾼다.
- **하는 일**: 5.5절 예시 ①을 자기 프로젝트의 포매터와 정적 분석기로 바꿔 등록한다. 일부러 규칙을 어긴 편집을 시켜 훅이 걸리는지, 모델이 사유를 받아 고치는지 본다.
- **완료 조건**: 편집 후 검사가 실제로 돌고, 실패 메시지에 고치는 법이 들어 있다.

### 랩 9. 서브에이전트

- **목표**: 컨텍스트를 격리하고 검증자를 분리한다.
- **하는 일**: 7.4절의 `pr-reviewer`처럼 읽기와 diff 조회만 가능한 검토자를 정의하고, 랩 4의 변경을 검토하게 한다. 메인 대화에 무엇이 돌아오는지 `/context`로 확인한다.
- **완료 조건**: 메인 컨텍스트에 diff 전체가 아니라 요약만 쌓였고, 검토자가 쓰기 도구를 갖지 않았다.

### 랩 10. 비용 측정

- **목표**: 줄였는지 재서 확인한다.
- **하는 일**: 같은 작업을 tool search 설정 전후로 한 번씩 돌린다. 매번 `/context`와 `/usage`를 적고, 조직 대시보드의 크레딧 차감과 대조한다.
- **완료 조건**: `MCP tools` 행의 변화와 작업당 크레딧 차이를 숫자로 말할 수 있다.

### 랩 11. 패키징

- **목표**: 자산을 두 번째 저장소로 옮긴다.
- **하는 일**: 랩 7~9의 산출물을 플러그인 한 디렉터리로 묶고 다른 저장소에 설치한다.
- **완료 조건**: 새 저장소에서 스킬과 서브에이전트가 보이고, 훅이 돈다.

### 랩 12. (선택) 첫 루프

- **목표**: 사람이 매 턴 프롬프트하지 않는 구간을 하나 만든다.
- **하는 일**: 부록 C.5를 읽고, 되돌리기 쉬운 작업 하나에 완료 조건을 쓴다. 조건에는 측정 가능한 종료 상태, 그것을 증명할 검사, 건드리면 안 되는 것, 턴 상한을 담는다.
- **완료 조건**: 조건이 참이 되어 스스로 멈췄거나, 상한에 걸려 멈췄다. 토큰 소모를 확인했다.

---

## 부록 C. 심화 읽기

### C.1 네 층의 포함 관계 논쟁

2.2절은 프롬프트 → 컨텍스트 → 하네스 → 루프의 이동만 다루고 포함 관계는 다루지 않았다. `프롬프트 ⊂ 컨텍스트 ⊂ 하네스 ⊂ 루프`라는 도식이 흔히 쓰이는데, 단계마다 근거의 강도가 다르다.

| 관계 | 상태 | 근거 |
| --- | --- | --- |
| 프롬프트 ⊂ 컨텍스트 | 확정 | Anthropic이 컨텍스트 엔지니어링을 프롬프트 엔지니어링의 자연스러운 진화로 규정하고, 정의에 "프롬프트 밖에서 흘러들어오는 모든 정보"를 포함시킨다 |
| 컨텍스트 ⊂ 하네스 | 지배적이지만 만장일치는 아니다 | 2026년의 여러 정리가 "각 분야가 앞 분야를 포함하며 경계마다 통제 가능한 표면이 넓어질 뿐 대체가 아니다"라고 쓴다. 반대로 셋을 병렬 직무로 보는 관점도 뚜렷하다 |
| 하네스 ⊂ 루프 | 명명자 본인의 규정 | Osmani가 루프를 "타이머로 돌아가고, 헬퍼를 띄우고, 스스로에게 일을 먹이는 하네스"로 정의한다 |

그래서 이 세미나는 포함 관계를 단정하지 않고, 각 층이 **답하는 질문**으로 구분한다. 어떤 입장을 택하든 흔들리지 않는 구분선이기 때문이다.

| 층 | 답하는 질문 | 한 단어 |
| --- | --- | --- |
| 프롬프트 | 말을 어떻게 쓰는가 | 표현 |
| 컨텍스트 | 무엇이 들어가는가 | 입력 |
| 하네스 | 장치 전체를 어떻게 통제하는가. 어떤 도구를 부를 수 있고, 무엇을 건드려도 되고, 언제 멈추는가 | 통제 |
| 루프 | 그 통제를 무엇이 언제 돌리는가 | 주기 |

실무에서 자주 들리는 두 문장도 절반만 맞다.

- **"프롬프트 엔지니어링은 시스템 프롬프트 안으로 들어가 버렸다."** 두 갈래로 갈라졌다. 역할 부여, 출력 형식, 예시 제시 같은 **기법**은 하네스의 상시 텍스트(시스템 프롬프트, 지침 파일, 스킬 본문, 도구 설명)로 이주했다. 무엇을 시킬지와 성공 기준을 정하는 **과제 정의**는 여전히 사람이 매 턴 쓴다. 이 세미나가 12장에서 다루는 것이 후자다.
- **"컨텍스트 엔지니어링은 하네스의 일부다."** 집행 수단의 관점에서는 대체로 맞다. 주입 컨텍스트는 하네스가 조립하고, 수집 컨텍스트의 채널도 하네스가 등록한 도구 목록이 정하며, 압축과 격리도 하네스 기능이다. 그러나 등치는 아니다. 자료를 어떻게 준비해 둘 것인가(문서 신선도, 요약 설계)는 하네스 밖에 남고, 권한과 승인 경계와 종료 조건은 컨텍스트 밖에 있다.

한 줄 비유를 다시 적으면 이렇다. 모델은 CPU, 컨텍스트는 RAM, 하네스는 운영체제, 루프는 스케줄러다.

### C.2 MCP stateless 스펙 (2026-07-28)

본문 6.1절은 MCP의 구조만 다뤘다. 사내에서 MCP 서버를 운영할 사람에게는 2026년 7월 28일 스펙 개정이 중요하다. **프로토콜 코어가 stateless로 바뀌었다.**

| 제거된 것 | 제안 번호 |
| --- | --- |
| `initialize` / `initialized` 핸드셰이크 | SEP-2575 |
| `Mcp-Session-Id` 헤더 | SEP-2567 |

이제 모든 요청이 자기 완결적이다. 얻은 것은 네 가지다.

- **수평 확장**: 평범한 라운드로빈 로드밸런서 뒤에 두면 된다. sticky 규칙이 필요 없다.
- **장애 내성**: 인스턴스가 죽어도 다음 요청이 다른 인스턴스로 간다. 공유 세션 저장소가 필요 없다.
- **표준 HTTP 인프라**: 헤더 기반 라우팅, 게이트웨이, 캐시를 그대로 쓴다.
- **캐시 힌트**: 목록과 리소스 읽기 응답에 수명(`ttlMs`)과 범위(`cacheScope`)가 붙고(SEP-2549), 목록이 결정적 순서로 돌아와 상위의 prompt caching 적중률이 올라간다(4.10절과 직결된다).

서버가 사용자에게 되묻는 기능(elicitation, sampling)은 열린 스트림 대신 여러 번의 왕복으로 뒤집혔다. 서버가 "입력이 필요하다"는 결과와 상태 토큰을 돌려주고, 클라이언트가 사용자에게 물어 같은 도구를 응답과 함께 다시 호출한다(SEP-2322). 상태 관리 책임이 전송 계층에서 애플리케이션 계층으로 옮겨가면서 인가 쪽도 강화됐다. 인가 서버가 `iss` 파라미터를 돌려주고 클라이언트가 그것을 검증하게 한 것(SEP-2468, RFC 9207)이 그 예다.

**사내 시사점**: MCP 서버를 평범한 HTTP 서비스처럼 배포·운영할 수 있다. 마이그레이션 판단 기준은 하나다. 인스턴스가 여러 개이거나 수평 확장이 필요하면 전환할 가치가 있고, 단일 프로세스로 한 클라이언트만 상대하는 로컬 STDIO 서버라면 서두를 이유가 없다.

### C.3 MCP 비용 계산 모델

4.5절과 4.8절의 수치를 만든 모델이다. **측정값이 아니라 아래 가정에 따른 계산값이다.**

| 항목 | 값 |
| --- | --- |
| 구성 A (필수만) | 기본 컨텍스트 40K (1M 윈도우의 4%) |
| 구성 B (A + 선택 MCP) | 기본 컨텍스트 120K (12%) |
| 작업 중 컨텍스트 증가량 | 260K |
| 작업 1건당 모델 호출 수 | 40회 |
| 작업 1건당 출력 | 60K |
| 단가 (Opus 5.5, 100만 토큰당) | 캐시 읽기 $0.20, 캐시 쓰기(5분) $5, 출력 $20 |
| 작업 시작 시 캐시 | 절반은 5분 이내에 이어서 시작(유효), 절반은 5분 초과(만료) |

```python
P_READ, P_WRITE, P_OUT = 0.20, 5.00, 20.00   # $/1M tokens
GROWTH, CALLS, OUT = 260_000, 40, 60_000

def task_cost(base, cold_start, long_tool_misses=0):
    step = GROWTH / CALLS
    reads = writes = 0.0
    for i in range(1, CALLS + 1):
        prefix = base + (i - 1) * step        # 이번 호출 직전까지의 컨텍스트
        if i == 1:
            if cold_start: writes += base     # 5분 초과 → 고정 영역을 다시 쓴다
            else:          reads  += base     # 5분 이내 → 캐시에서 읽는다
        elif i <= 1 + long_tool_misses:
            writes += prefix                  # 5분 넘는 도구 실행 → prefix 전체를 다시 쓴다
        else:
            reads += prefix
    writes += GROWTH                          # 새로 붙는 부분은 한 번씩 쓴다
    return (reads * P_READ + writes * P_WRITE + OUT * P_OUT) / 1e6

def avg(base, cold_ratio=0.5, long_tool_misses=0):
    return (cold_ratio * task_cost(base, True, long_tool_misses)
            + (1 - cold_ratio) * task_cost(base, False, long_tool_misses))

print(avg(40_000), avg(120_000))   # 3.93, 4.76
```

**기본 결과.** 기본 컨텍스트가 윈도우의 1%(10K)씩 늘 때마다 작업당 약 $0.10이 늘어난다.

| 기본 컨텍스트 | 작업당 비용 |
| --- | --- |
| 4% (40K) | $3.93 |
| 8% (80K) | $4.35 |
| 12% (120K) | $4.76 |
| 20% (200K) | $5.59 |
| 30% (300K) | $6.63 |

**민감도.** 차이를 벌리는 것은 호출 수와 캐시 만료 횟수다.

| 변수 | A (4%) | B (12%) | 차이 |
| --- | --- | --- | --- |
| 호출 20회 | $3.25 | $3.76 | +$0.51 (+16%) |
| 호출 40회 | $3.93 | $4.76 | +$0.83 (+21%) |
| 호출 80회 | $5.29 | $6.76 | +$1.47 (+28%) |
| 만료 비율 0% | $3.83 | $4.47 | +$0.64 (+17%) |
| 만료 비율 100% | $4.03 | $5.05 | +$1.02 (+25%) |
| 작업 중 5분 넘는 도구 실행 1회 | $4.15 | $5.37 | +$1.22 |

```text
작업당 차이 ≈ 추가 컨텍스트 × 캐시 읽기 단가 × 호출 수 + 추가 컨텍스트 × 캐시 쓰기 단가 × 만료 횟수
            = 80K × $0.20/M × N                    + 80K × $5.00/M × (만료 횟수)
            = $0.016 × N                           + $0.40 × (만료 횟수)
```

**tool search를 켠 경우(4.8절의 표).** 선택 MCP 80K를 미루면 평소에는 도구 이름 약 3K만 실린다고 보아 기본 컨텍스트를 43K로 놓았다. 쓰는 경우는 작업 중간(20번째 호출)부터 도구 두 개(4K)가 더 실린다고 놓았다.

| 구성 | 기본 컨텍스트 | 작업당 비용 |
| --- | --- | --- |
| A: 필수만 | 40K | $3.93 |
| B: 필수 + 선택 MCP, tool search 없음 | 120K | $4.76 |
| B + tool search, 이번 작업에서 쓰지 않음 | 43K | $3.96 |
| B + tool search, 도구 두 개를 씀 | 43K → 47K | $4.00 |

**이 모델의 한계.** ① 캐시가 정상적으로 적중한다고 가정한다. 적중하지 않으면 고정 영역의 반복 비용이 훨씬 커지므로, 줄이는 효과도 계산보다 커진다. ② 호출마다 컨텍스트가 균일하게 늘어난다고 가정한다. 실제로는 큰 파일을 읽는 호출에서 몰린다. ③ Opus 5.5 단가 기준이다. 모델을 바꾸면 캐시 읽기와 입력의 배수가 달라진다. 캐시 읽기가 기본 입력의 0.05배인 모델(Opus 5.5, Sonnet 5.5)과 0.1배인 모델에서 결과가 다르다. ④ 크레딧 환산은 1 크레딧 = $0.01이다.

### C.4 eval: 하네스 변경의 효과 측정

14.7절은 3겹 구조의 효과를 반복 지적 비율과 되돌림 건수로 재라고 했다. 더 일반적으로 "지침을 고치고 스킬을 추가하고 모델을 바꿨는데 좋아졌는가"를 재는 방법이 eval이다. 감으로 답하면 하네스 엔지니어링이 아니라 미신이다.

**최소 세트**는 다음 셋으로 만든다.

1. 실제 저장소에서 뽑은 대표 과제 5~10개.
2. 과제마다 **자동으로 판정되는 성공 조건**(테스트 통과, 특정 심볼 존재, 정적 분석 무경고).
3. 각 과제 3회 반복. 모델은 비결정적이다.

| 지표 | 뜻 |
| --- | --- |
| 1회 통과율 | 사람의 개입 없이 성공한 비율 |
| 개입 횟수 | 사람이 방향을 고쳐 준 횟수 |
| 크레딧 소모 | 과제당 실제 비용 |
| 소요 시간 | 벽시계 기준 |
| 회귀 발생 | 기존 테스트를 깬 비율 |

eval이 가장 먼저 쓸모를 보이는 자리가 **지침 다이어트**다(5.2절). 지시를 한 묶음 지운 뒤 같은 과제 세트를 다시 돌리면, 줄인 것이 손해였는지 아닌지가 숫자로 나온다. 두 회사가 공개한 수치(80% 삭제에도 측정 가능한 손실 없음, 얇은 프롬프트에서 점수 10~15% 상승)도 모두 자사 평가 세트로 잰 값이다. 같은 방법을 우리 저장소에서 쓰는 것이 이 절의 내용이다.

판정은 생성자와 분리한다(6.3절). 그리고 한 번 일어난 실패는 eval 세트에 추가한다. eval을 CI에 넣어 지침과 플러그인이 바뀔 때 자동으로 돌리면, 2.4절의 제1원칙이 조직 단위로 구현된다.

### C.5 루프 엔지니어링 실전

2.2절에서 지평선으로만 소개한 루프다. 개론의 범위를 넘으므로 여기에 둔다. **먼저 전제를 확인한다.** 아래 셋이 없으면 이르다.

| 전제 | 확인 질문 | 관련 절 |
| --- | --- | --- |
| 결정적 검증 | 성공과 실패를 사람 없이 판정할 수 있는가 | 14.3, C.4 |
| 빠른 피드백 층 | 그 판정이 분 단위로 끝나는가 | 14.3 |
| 하네스 | 지침·훅·스킬이 팀 자산으로 관리되는가 | 5~7장 |

빌드가 20분인 상태에서 루프를 돌리면 하룻밤에 몇십 사이클밖에 돌지 못한다. 호스트 빌드 층을 만드는 일이 루프 도입의 실질적 선행 과제다.

**도구가 이미 제공하는 것.** 세션을 계속 돌리는 방법은 셋이고, 다음 턴을 무엇이 시작시키는지가 다르다(출처: [Claude Code 문서, Keep Claude working toward a goal](https://code.claude.com/docs/en/goal), 2026-10 확인).

| 방법 | 다음 턴이 시작되는 때 | 멈추는 때 |
| --- | --- | --- |
| `/goal` | 앞 턴이 끝났을 때 | 평가자가 조건 충족 또는 달성 불가로 판정했을 때, 사람이 고쳐야 하는 오류가 났을 때, `/goal clear` |
| `/loop` | 정해진 시간 간격이 지났을 때 | 사람이 멈추거나, Claude가 끝났다고 판단할 때 |
| Stop 훅 | 앞 턴이 끝났을 때 | 내가 쓴 스크립트나 프롬프트가 정하는 때 |

`/goal`은 세션 스코프의 프롬프트형 Stop 훅을 감싼 것이다. 턴이 끝날 때마다 조건과 대화를 작고 빠른 모델(평가자)에 보내 충족·미충족·달성 불가 가운데 하나를 받는다. **코드를 쓴 모델이 자기 완료 여부를 채점하지 않는다**는 점이 설계의 핵심이다(6.3절의 생성자·검증자 분리가 정지 조건에 적용된 형태다).

**완료 조건 쓰는 법.** 평가자는 도구를 부르지 않는다. **대화에 드러난 것만 보고 판정한다.** 그래서 조건은 에이전트의 출력으로 증명될 수 있는 형태여야 한다. 공식 문서가 드는 요소는 셋이고, 여기에 상한을 더한다.

1. 측정 가능한 종료 상태 (테스트 결과, 빌드 종료 코드, 파일 수, 빈 대기열)
2. 그것을 어떻게 증명할지 (실행할 검사)
3. 건드리면 안 되는 것에 대한 제약
4. 턴 또는 시간 상한 (조건 안에 "또는 20턴 뒤 멈춘다"처럼 적는다)

```text
/goal test/auth 하위 모든 테스트가 통과하고 clang-tidy 경고가 0이 될 것.
      ./run_tests.sh unit 과 ./scripts/tidy.sh 로 증명할 것.
      public 헤더의 시그니처는 변경하지 말 것. 30턴 이내.
```

`/goal the app is production-ready` 같은 조건은 실패한다. "production-ready"를 증명하는 명령이 없으므로 평가자가 대조할 대상이 없고, 루프는 토큰만 태우며 맴돈다.

**종료 설계.** 루프에는 독립적인 출구가 여러 개 있어야 한다. 도구가 내장한 것과 내가 세워야 하는 것을 구분한다.

| 출구 | 어떻게 세우나 |
| --- | --- |
| 성공 (조건 충족) | 내장. 평가자가 판정한다 |
| 달성 불가 판정 | 내장. 평가자가 조건을 충족시킬 수 없다고 보면 목표가 해제된다 |
| 치명적 오류 (자격증명 실패, 크레딧 소진, 컨텍스트 초과, 모델 없음) | 내장. 목표가 해제되고 사유가 표시된다 |
| 무진전 (여러 턴 동안 도구를 쓰지 않음) | 내장. 경고를 내고 사람에게 제어를 돌려준다 |
| 반복 상한 | 대화형 세션에서는 조건 안에 턴 상한을 적는다(평가자가 대화를 보고 판정하므로 하드 캡은 아니다). 비대화형 실행에서는 `claude -p --max-turns N`이 하드 캡이다 |
| 토큰·금액 예산 | 비대화형 실행에서는 `claude -p --max-budget-usd 5.00`으로 건다. 클라이언트의 추정치 기준이고 상한을 넘을 수 있으므로 여유를 둔다. 서브에이전트의 소모도 합산된다. 대화형 세션에는 적용되지 않으므로 래퍼 스크립트로 세운다 |

**오류를 두 종류로 구분하는 것이 핵심이다.** 실패한 테스트는 행동할 재료이고, 없는 자격증명은 즉시 정지 사유다. 이 둘을 구분하지 못하는 루프는 자격증명을 찾겠다며 밤새 예산을 태운다.

**실패 모드.**

| 실패 | 증상 | 대응 |
| --- | --- | --- |
| 컨텍스트 부패 | 길어질수록 품질이 조용히 떨어진다 | 압축, 가지치기, 서브에이전트 격리 |
| 무진전 반복 | 같은 실패를 되풀이한다 | 무진전 감지와 하드 캡 |
| 보상 해킹 | 실패하는 테스트를 지워 검사를 통과시킨다 | 의도를 담은 종료 기준, 테스트 수정에 사람 게이트(13.4절) |
| 환각된 성공 | 검증 없이 "완료"를 보고한다 | 자기 보고 대신 결정적 검증만 신뢰한다(10.5절) |
| 오류 누적 | 초기 실수가 전체 궤적으로 번진다 | 끝이 아니라 매 스텝 검증 |
| 비용 폭증 | 조용히 토큰을 태운다 | 예산 가드와 prompt caching |

보상 해킹은 차량 소프트웨어에서 특히 위험하다. "CI를 초록으로"를 목표로 준 루프는 CI를 초록으로 만들 뿐, 코드를 옳게 만들지 않는다.

**패턴과 적용 후보.** 동작하는 가장 단순한 패턴을 쓴다. 디버깅할 수 없는 정교한 멀티 에이전트 시스템보다, 결정적 검증자를 가진 단일 ReAct 루프가 낫다.

| 패턴 | 구조 | 언제 |
| --- | --- | --- |
| ReAct | 추론 → 행동 → 관찰 반복 | 기본형 |
| Reflexion | 행위자 + 평가자 + 자기성찰 메모 | 세션 안에서 시도가 나아지길 원할 때 |
| Plan-and-Execute | 계획자와 실행자 분리 | 길고 다단계인 작업의 드리프트 방지 |
| Evaluator-Optimizer | 생성자와 평가자가 기준 통과까지 왕복 | 합격 기준을 명확히 쓸 수 있을 때 |
| Orchestrator-Workers | 오케스트레이터가 분해·위임·종합 | 병렬 대규모 작업 (9.4절이 이 형태다) |

| 후보 | 적합도 | 이유 |
| --- | --- | --- |
| 야간 CI 실패 트리아지 | 높다 | 성공 조건이 명확하고 되돌리기 쉽다 |
| 빌드 레시피 버전 범프 + 호스트 빌드 검증 | 높다 | 반복적이고 결정적으로 검증된다 |
| 정적 분석 경고 일괄 정리 | 높다 | 판정이 기계적이다 |
| 신규 기능 구현 | 낮다 | 종료 조건을 쓸 수 없다 |
| 타이밍·하드웨어 이슈 디버깅 | 부적합 | 검증이 HIL에 묶여 있다 |

첫 루프는 반드시 되돌리기 쉬운 것으로 고른다. 그리고 8.2절의 결론을 다시 적어 둔다. 루프는 일을 바꿀 뿐 그 일에서 사람을 지우지 않는다.

### C.6 임베디드·차량 적용 매트릭스

본문은 도메인 예시를 장마다 흩어 놓았다. 어디서부터 적용할지는 두 축으로 정한다. **검증이 쉬운가, 반복적인가**다.

| | 검증 쉬움 | 검증 어려움 |
| --- | --- | --- |
| **반복적** | **1순위**: 빌드 레시피, ARXML·IDL 설정, 설정 파일, 단위 테스트, 보일러플레이트 | 3순위: 빌드 스크립트, CI 설정 |
| **비반복적** | 2순위: 리팩터링, 로그 분석, 문서화, 마이그레이션 | **최후**: 타이밍 이슈, 하드웨어 디버깅, 성능 튜닝 |

1순위부터 시작한다. 첫 성공 경험을 어디서 만드느냐가 도입의 성패를 가른다.

우리 도메인의 네 제약과 대응도 한 표로 모은다.

| 제약 | 결과 | 대응 | 본문 |
| --- | --- | --- | --- |
| 빌드가 느리다 | 에이전트의 고치고 확인하는 반복이 성립하지 않는다 | 검증을 층으로 나누고 빠른 층부터 만든다. `compile_commands.json`을 제공한다 | 14.3 |
| 타깃 하드웨어가 필요하다 | 검증을 자동화할 수 없다 | 모의 HAL과 스텁으로 로직 계층만 호스트에서 시험한다. 에뮬레이션, 정적 분석 | 14.3 |
| 규격 준수 (MISRA, 기능안전) | 잘못된 코드의 비용이 극단적이다 | 체커를 편집 후 훅으로. 안전 관련 경로에는 전용 룰과 전용 리뷰 | 5.5, 13.4 |
| 폐쇄망 | 외부 도구와 모델의 선택지가 제한된다 | 사내 MCP 서버(C.2), 사내 마켓플레이스, 외부 문서 미러링 | 3.5, 6.4 |

---

## 부록 D. 용어집

0장의 용어 통일표를 포함해 이 세미나가 쓰는 용어를 모았다. 괄호 안은 처음 나오는 절이다.

| 용어 | 정의 |
| --- | --- |
| 에이전트 루프 (1.1) | 모델 추론 → 도구 호출 → 하네스 실행 → 결과 주입의 반복. 이 반복이 있는 것이 에이전트다 |
| 하네스 (1.1, 2.1) | 에이전트에서 모델을 뺀 전부. 컨텍스트를 조립하고 도구를 실행하는 프로그램과 그 주변 장치 |
| 빌더 하네스 (2.1) | 제작사가 제품에 내장한 하네스. 시스템 프롬프트, 내장 도구, 코드 검색, 오케스트레이션. 고르는 대상이다 |
| 사용자 하네스 (2.1) | 우리 팀이 우리 코드베이스에 맞춰 덧대는 하네스. 지침, Skill, Hook, MCP, 권한 설정. 만드는 대상이다 |
| 컨텍스트 윈도우 (1.3) | 모델이 한 번의 호출에서 볼 수 있는 입력 전체. 에이전트의 작업 기억 |
| 주입 컨텍스트 (1.3) | 하네스가 요청을 시작할 때 조립해 넣는 것. 시스템 프롬프트, 지침, 도구 정의 |
| 수집 컨텍스트 (1.3) | 모델이 루프 안에서 도구로 가져온 것. 읽은 파일, 검색 결과, 빌드 로그 |
| 고정 영역 / 누적 영역 (1.4) | 호출마다 크기가 그대로인 부분 / 호출마다 커지는 부분. 비용과 품질 저하의 뿌리 |
| 압축 (compaction) (1.3) | 컨텍스트가 찼을 때 하네스가 오래된 도구 출력과 대화를 요약으로 바꾸는 것 |
| 가이드 / 센서 (2.3) | 행동 전에 방향을 잡아 주는 장치 / 행동 후에 결과를 관찰해 고치게 하는 장치 |
| 계산적 센서 / 추론적 센서 (2.3) | 결정적으로 판정하는 검증 장치(테스트, 린터, 정적 분석) / 모델이 해석해 판정하는 장치(리뷰 에이전트, LLM 심사) |
| 과잉 제약 (2.5) | 촘촘한 규칙과 반복 강조가 최신 모델의 판단을 방해해 토큰과 사고 시간을 더 쓰게 만드는 상태 |
| 지침 파일 (5.2) | 세션을 시작할 때 읽히는 프로젝트 상수. `AGENTS.md`(공개 형식)와 `CLAUDE.md`(Claude Code 형식) |
| 조건부 룰 (5.3) | `paths:`로 경로를 지정해, 그 경로의 파일을 다룰 때만 로드되는 지침 |
| Skill (5.4) | 지식과 절차를 담은 `SKILL.md`. 평소에는 설명 한 줄, 쓸 때만 본문이 실린다 |
| Hook (5.5) | 정해진 이벤트에서 하네스가 자동으로 실행하는 명령. 요청이 아니라 강제다 |
| MCP (6.1) | Model Context Protocol. 에이전트와 외부 도구·데이터를 잇는 공개 규격 |
| Tools / Resources / Prompts (6.1) | MCP 서버가 내놓는 것. 모델이 부르는 함수 / 호스트가 붙이는 데이터 / 사용자가 고르는 템플릿 |
| 간접 프롬프트 인젝션 (6.2) | 도구가 읽어 온 글에 심긴 지시를 모델이 따르게 만드는 공격 |
| tool search (defer loading) (4.8) | 코딩 에이전트가 도구 정의를 필요할 때만 컨텍스트에 싣는 기능. MCP 문서의 progressive discovery(catalog → inspect → execute)가 같은 원리다 |
| 서버 쪽 재설계 (4.6) | MCP 서버가 메타 도구 몇 개만 노출하도록 서버를 다시 만드는 방식. tool search와 구분해 부른다 |
| prompt caching (4.10) | 같은 앞부분을 캐시해 할인 단가로 계산하는 기능. API 키 방식의 기본 수명은 5분이다 |
| Subagent (6.3) | 격리된 자기 컨텍스트에서 일하고 요약만 돌려주는 일꾼 |
| Plugin (6.4) | 부품을 묶은 배포 단위 |
| 권한 모드 (7.5) | 에이전트가 묻지 않고 할 수 있는 행동의 범위. Manual(`default`), `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| Plan mode (7.5) | 조사와 계획까지만 하고, 사람이 승인한 뒤에 편집을 시작하는 모드 |
| HITL (8.1) | Human in the Loop. 자동화된 흐름의 정해진 지점에 사람의 판단을 넣는 것 |
| 개입 지점 4단계 (8.1) | 요청 → 계획 → 실행 → 검증. 사람이 개입할 수 있는 네 지점 |
| Context Rot (9.3) | 컨텍스트의 토큰이 늘어날수록 그 안의 정보를 정확히 꺼내 쓰는 능력이 떨어지는 현상 |
| 환각 (10.1) | 그럴듯하지만 사실이 아닌 진술 |
| 지식 컷오프 (10.1) | 모델의 학습 데이터가 끝나는 시점. 그 뒤의 변화는 모른다 |
| 사전학습 / 지도 파인튜닝 / 강화학습 / 평가 (10.2) | 모델이 만들어지는 네 단계 |
| RLHF (10.2) | 사람이 고른 선호를 보상으로 쓰는 강화학습 |
| 자동 채점(RLVR) (10.2) | 프로그램이 판정한 결과를 보상으로 쓰는 강화학습 |
| 들쭉날쭉한 지능 (11.1) | 검증 가능한 영역에서는 정점을 찍고 그 밖에서는 거친 채로 남는 능력의 분포 |
| Verifier's Rule (11.2) | 과제를 가르치기 쉬운 정도는 그 과제가 얼마나 검증 가능한가에 비례한다는 경험 법칙 |
| 보상 해킹 (11.2) | 점수만 올리는 꼼수. 목적은 이루지 않은 채 채점만 통과하는 답을 내는 것 |
| 과소명세 (12.1) | 요청에 필요한 정보가 빠져 있거나 모호한 상태 |
| 양방향 문 / 일방통행 문 (13.2) | 싸게 되돌릴 수 있는 결정 / 되돌릴 수 없거나 비용이 큰 결정 |
| 3겹 구조 (14.2) | 자동 체크 → 자동 리뷰 → 사람 리뷰. 사람 앞에 두 겹을 세우는 PR 흐름 |
| Merge Danger (14.5) | PR에 문 유형과 영향 범위를 적어 리뷰 깊이를 정하는 표기 |
| 이해 부채 (15.3) | 존재하는 코드와 내가 실제로 아는 것 사이의 간극 |
| eval (C.4) | 하네스 변경의 효과를 재현 가능하게 측정하는 과제 세트 |
| 루프 엔지니어링 (2.2, C.5) | 하네스를 무엇이 언제 돌리고 언제 멈추는가를 설계하는 층 |

---

## 부록 E. 참고 자료

장 번호 순으로 모았다. 괄호 안은 확인 시점이다. 공식 문서는 자주 바뀌므로, 인용한 문장이 아직 그대로인지 발표 전에 다시 연다.

### 0~2장: 원리와 하네스

- Claude API 문서, Tool use with Claude — `tool_use`/`tool_result` 규격과 도구 정의의 과금: <https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview> (2026-10)
- Claude Code 문서, Explore the context window — 윈도우 크기와 압축 뒤에 남는 것: <https://code.claude.com/docs/en/context-window> (2026-10)
- Claude Code 문서, How Claude Code works: <https://code.claude.com/docs/en/how-claude-code-works> (2026-10)
- Claude Code 문서, Prompt caching — 매 호출 전체 재전송: <https://code.claude.com/docs/en/prompt-caching> (2026-10)
- Claude Code 문서, Debug your configuration — `/context`의 범주: <https://code.claude.com/docs/en/debug-your-config> (2026-10)
- GitHub Docs, Copilot CLI command reference: <https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference> (2026-10)
- Birgitta Böckeler, Harness engineering for coding agent users — `Agent = Model + Harness`, 세 고리, 가이드와 센서, 제1원칙: <https://martinfowler.com/articles/harness-engineering.html> (2026-04-02)
- Anthropic, Effective context engineering for AI agents: <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents> (2025-09-29)
- Addy Osmani, Loop Engineering — 루프의 명명, Steinberger·Cherny 인용: <https://addyosmani.com/blog/loop-engineering/> (2026-06-07)
- Anthropic, The new rules of context engineering for Claude 5 generation models — 시스템 프롬프트 80% 삭제와 여섯 가지 전환: <https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models> (2026-07-24)
- OpenAI, Prompting guidance for GPT-5.6 Sol — "Simplify prompts first": 반복 지시·예시 삭제와 도구 설명 단순화, 얇은 프롬프트의 내부 평가 수치(점수 +10~15%, 토큰 −41~66%, 비용 −33~67%, "directional"), 충돌하는 규칙의 위험, `ALWAYS`/`NEVER`는 불변에만, 한 묶음씩 지우고 같은 eval 재실행: <https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6> (2026-10)
- OpenAI, Latest model prompting best practices (GPT-6 계열) — 더 긴 지시를 더 잘 따르지만 컨텍스트에 더 민감하다, 스킬과 `AGENTS.md` 감사 권고, 스킬 파일의 충돌이 작업을 멈추게 할 수 있음, Codex 이전 명령: <https://developers.openai.com/api/docs/guides/latest-model> (2026-10)

### 3장: 도구 선택

- VS Code 문서, Understand agent harnesses: <https://code.visualstudio.com/docs/agents/concepts/agent-harnesses> (2026-10)
- VS Code 문서, AI settings reference — `chat.agent.maxRequests`: <https://code.visualstudio.com/docs/agents/reference/ai-settings> (2026-10)
- GitHub Changelog, GitHub Copilot now supports OpenCode: <https://github.blog/changelog/2026-01-16-github-copilot-now-supports-opencode/> (2026-01-16)
- GitHub Docs, Requests in GitHub Copilot (legacy) — 요청 수 과금에서 도구 호출이 세어지지 않았다는 원문: <https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests> (2026-10)
- GitHub Docs, Hooks (Copilot CLI): <https://docs.github.com/en/copilot/concepts/agents/hooks> (2026-10)
- Claude Code 문서, Memory — `AGENTS.md` 읽기 조건과 버전: <https://code.claude.com/docs/en/memory#agents-md> (2026-10)
- Claude Code 문서, Skills — 표준 필드와 전용 기능의 구분: <https://code.claude.com/docs/en/skills> (2026-10)
- MCP 블로그, MCP joins the Agentic AI Foundation: <https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/> (2025-12-09)
- Agent Skills 공개 규격: <https://agentskills.io/> (2026-10)
- Agent Plugins 1.0과 호환 클라이언트 목록: <https://agent-plugins.org/> · <https://agent-plugins.org/compatible-clients> (2026-10)
- opencode 문서: <https://opencode.ai/docs/> (2026-10)
- oh-my-openagent 저장소: <https://github.com/code-yeongyu/oh-my-openagent> (2026-10)
- Codex 설정 참조 — `wire_api`와 커스텀 프로바이더: <https://learn.chatgpt.com/docs/config-file/config-reference> (2026-10)

### 4장: 비용

- GitHub Blog, GitHub Copilot is moving to usage-based billing — 2026-06-01 전환, 플랜별 포함 크레딧, 프로모션, 폴백 종료: <https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/> (2026-04-27)
- GitHub Docs, Usage-based billing for organizations and enterprises — Business 1,900 크레딧, 청구 단위 합산, 이월 없음, 추가 사용 기본 켜짐, 저가 모델 자동 폴백 없음: <https://docs.github.com/en/copilot/concepts/billing-and-usage/organizations-and-enterprises/billing> (2026-10)
- GitHub Docs, Models and pricing for GitHub Copilot — 모델별 입력·캐시·출력 단가: <https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing> (2026-10)
- Claude 가격 문서: <https://platform.claude.com/docs/en/about-claude/pricing> (2026-10-12)
- Claude Code 문서, Manage costs effectively — 모델 선택, 지침을 스킬로 옮기기, MCP 오버헤드 줄이기, `/usage`: <https://code.claude.com/docs/en/costs> (2026-10)
- Claude Code 문서, Prompt caching — 캐시 수명과 무효화: <https://code.claude.com/docs/en/prompt-caching#cache-lifetime> (2026-10)
- Claude API 문서, Prompt caching — 수명을 요청 시작 시점부터 잰다: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching> (2026-10)
- MCP 스펙 2026-07-28, Tools — `tools/list`와 `tools/call`: <https://modelcontextprotocol.io/specification/2026-07-28/server/tools> (2026-10)
- MCP 문서, Client Best Practices — progressive discovery, 호스트가 정의 주입을 미룬다, 플랫폼의 tool search 우선: <https://modelcontextprotocol.io/docs/2026-07-28/develop/clients/client-best-practices> (2026-10)
- Claude Code 문서, Scale with MCP tool search / Configure tool search: <https://code.claude.com/docs/en/mcp#scale-with-mcp-tool-search> · <https://code.claude.com/docs/en/mcp#configure-tool-search> (2026-10)
- Agent SDK 문서, Tool search: <https://code.claude.com/docs/en/agent-sdk/tool-search> (2026-10)
- Claude API 문서, Tool search tool — 도구 30~50개 이상에서 선택 정확도 저하: <https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool> (2026-10)
- Anthropic, Introducing advanced tool use — 다섯 서버 58개 도구 약 55K 토큰, MCP 평가 정확도 변화: <https://www.anthropic.com/engineering/advanced-tool-use> (2025-11-24)
- Anthropic, Writing effective tools for agents: <https://www.anthropic.com/engineering/writing-tools-for-agents> (2025-09-11)
- Anthropic, Code execution with MCP — `search_tools` 방식: <https://www.anthropic.com/engineering/code-execution-with-mcp> (2025-11-04)
- GitHub Docs, Loading tools on demand with tool search (Copilot CLI): <https://docs.github.com/en/copilot/concepts/agents/copilot-cli/tool-search> (2026-10)
- opencode 문서, MCP servers: <https://opencode.ai/docs/mcp-servers/> (2026-10)
- Codex 설정 참조 — `[mcp_servers.<id>]`, retired switches(`tool_search` 등): <https://learn.chatgpt.com/docs/config-file/config-reference> (2026-10)
- Codex 문서, Model Context Protocol — 서버 instructions를 서버 전체 안내로 사용, 앞 512자 권고: <https://learn.chatgpt.com/docs/extend/mcp> (2026-10)

### 5~7장: 확장 부품과 셋업

- Claude Code 문서, Extend Claude Code — 부품별 컨텍스트 비용, 요청과 강제의 차이, 도입 순서: <https://code.claude.com/docs/en/features-overview> (2026-10)
- Claude Code 문서, Audit your instruction files — `/doctor prompt-audit`의 검사 대상과 범위, v2.1.283 이상: <https://code.claude.com/docs/en/memory#audit-your-instruction-files> (2026-10)
- Claude Code 문서, Memory — 200줄 권고, import가 시작 때 로드된다는 문장, 백틱으로 감싸기: <https://code.claude.com/docs/en/memory#import-additional-files> · <https://code.claude.com/docs/en/memory#write-effective-instructions> (2026-10)
- Claude Code 문서, Best practices — 지침 작성, 세션 관리, 탐색→계획→구현, 적대적 리뷰: <https://code.claude.com/docs/en/best-practices> (2026-10)
- Claude Code 문서, Skills — `description` 1,536자 한도, 500줄 권고, 보조 파일 링크, `disable-model-invocation`: <https://code.claude.com/docs/en/skills> · <https://code.claude.com/docs/en/skills#add-supporting-files> (2026-10)
- Claude Code 문서, Hooks reference / Hooks guide — stdin JSON, 종료 코드, `permissionDecision`, `onFailure`: <https://code.claude.com/docs/en/hooks> · <https://code.claude.com/docs/en/hooks-guide> (2026-10)
- Claude Code 문서, Permissions — Bash 규칙의 한계: <https://code.claude.com/docs/en/permissions#bash-rule-limits> (2026-10)
- Claude Code 문서, Choose a permission mode — 여섯 모드, auto 모드와 분류기, 시작 모드: <https://code.claude.com/docs/en/permission-modes> (2026-10)
- Claude Code 문서, Create custom subagents — 격리 동작과 지침 재로드: <https://code.claude.com/docs/en/sub-agents> (2026-10)
- Claude Code 문서, Plugins overview: <https://code.claude.com/docs/en/plugins/overview> (2026-10)
- Claude Code 문서, Security — MCP 보안, 승인 피로, 네트워크 명령: <https://code.claude.com/docs/en/security> (2026-10)
- MCP 문서, Architecture overview / 스펙의 Server 개요 — Host·Client·Server, Tools·Resources·Prompts, 사람이 거부할 수 있어야 한다는 문장: <https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture> · <https://modelcontextprotocol.io/specification/2026-07-28/server> (2026-10)
- MCP 문서, Local Server Security: <https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/local-server-security> (2026-10)
- MCP 저장소, SECURITY.md — STDIO 명령 실행은 의도된 기능: <https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/SECURITY.md> (2026-10)
- Postmark 공지, 악성 `postmark-mcp` 패키지: <https://postmarkapp.com/blog/information-regarding-malicious-postmark-mcp-package> (2025-09-25)
- Straiker STAR Labs, Oura MCP 서버 복제 공급망 공격: <https://www.straiker.ai/blog/smartloader-clones-oura-ring-mcp-to-deploy-supply-chain-attack> (2026-02-05)
- OX Security, MCP STDIO 명령 주입 — 메인 글과 권고: <https://www.ox.security/blog/the-mother-of-all-ai-supply-chains-critical-systemic-vulnerability-at-the-core-of-the-mcp/> · <https://www.ox.security/blog/mcp-supply-chain-advisory-rce-vulnerabilities-across-the-ai-ecosystem/> (2026-04-15)
- VentureBeat, 20만 추정치가 외삽이라는 보도: <https://venturebeat.com/security/mcp-stdio-flaw-200000-ai-agent-servers-exposed-ox-security-audit> (2026-05-01)
- Simon Willison, The lethal trifecta for AI agents: <https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/> (2025-06-16)
- Anthropic, How we built our multi-agent research system — 토큰 약 15배, 코딩의 병렬화 한계: <https://www.anthropic.com/engineering/multi-agent-research-system> (2025-06-13)
- MCP Python SDK 문서: <https://py.sdk.modelcontextprotocol.io/> (2026-10)
- 사내망에서 공식 사이트에 접근할 수 없을 때의 미러: <https://github.com/posimind/claude-code-docs/blob/main/docs/memory.md> · <https://github.com/posimind/claude-code-docs/blob/main/docs/skills.md>

### 8~9장: HITL과 컨텍스트 한계

- Claude Code 문서, Choose a permission mode: <https://code.claude.com/docs/en/permission-modes> (2026-10)
- Addy Osmani, Loop Engineering — 루프가 사람을 지우지 않는다, 검증은 사람 몫: <https://addyosmani.com/blog/loop-engineering/> (2026-06-07)
- Cognition, Introducing SWE-grep and SWE-grep-mini — 첫 턴의 60% 이상이 컨텍스트 수집: <https://cognition.com/blog/swe-grep> (2025-10-16)
- You Wang, Michael Pradel, Zhongxin Liu, Are "Solved Issues" in SWE-bench Really Solved Correctly? — 7.8%, 29.6%, 28.6%, 6.2%p: <https://arxiv.org/abs/2503.15223> (ICSE 2026, arXiv v2 2025-09)
- Chroma, Context Rot: How Increasing Input Tokens Impacts LLM Performance: <https://research.trychroma.com/context-rot> (2025-07-14)
- Anthropic, Effective context engineering for AI agents — Context Rot, 서브에이전트 요약 규모: <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents> (2025-09-29)
- deep-code-audit README — N개 서브에이전트로 나눠 읽는 구조: <https://github.com/posimind/deep-code-audit> (2026-08-02 기준)

### 10장: 환각

- OpenAI, Why language models hallucinate — 정의, 채점 방식 변경 주장, 두 모델 비교 표: <https://openai.com/index/why-language-models-hallucinate/> (2025-09-05)
- Wei 외, Measuring short-form factuality in large language models (SimpleQA) — 표 3: <https://arxiv.org/abs/2411.04368> (2024-11)
- Vectara, Hallucination Leaderboard — 개편 후 수치: <https://github.com/vectara/hallucination-leaderboard> (2026-09-22 기준)
- Spracklen 외, We Have a Package for You! — 추천 패키지의 19.7%가 존재하지 않음: <https://arxiv.org/abs/2406.10279> (USENIX Security 2025)
- Kalai 외, Why Language Models Hallucinate: <https://arxiv.org/abs/2509.04664> (2025-09)
- Gekhman 외, Does Fine-Tuning LLMs on New Knowledge Encourage Hallucinations?: <https://arxiv.org/abs/2405.05904> (EMNLP 2024)
- OpenAI, GPT-4 Technical Report — 그림 8의 보정 오차: <https://arxiv.org/abs/2303.08774> (2023-03)
- Sharma 외, Towards Understanding Sycophancy in Language Models: <https://arxiv.org/abs/2310.13548> (2023-10)
- Song 외, The Hallucination Tax of Reinforcement Finetuning: <https://arxiv.org/abs/2505.13988> (2025-05)
- Ouyang 외, Training language models to follow instructions with human feedback: <https://arxiv.org/abs/2203.02155> (2022-03)
- Zhang 외, How Language Model Hallucinations Can Snowball: <https://arxiv.org/abs/2305.13534> (2023-05)
- Xu 외, Hallucination is Inevitable: <https://arxiv.org/abs/2401.11817> (2024-01)
- OpenAI o3 and o4-mini System Card — 표 4: <https://cdn.openai.com/pdf/2221c875-02dc-4789-800b-e7758f3722c1/o3-and-o4-mini-system-card.pdf> (2025-04-16)
- Artificial Analysis, AA-Omniscience: <https://artificialanalysis.ai/evaluations/omniscience> · 2차 집계: <https://suprmind.ai/hub/ai-hallucination-rates-and-benchmarks/> (2026-10-08)
- Andrej Karpathy, 2025 LLM Year in Review — 학습 단계 정리: <https://karpathy.bearblog.dev/year-in-review-2025/> (2025-12-19)

### 11장: 검증 가능성

- Jason Wei, Asymmetry of verification and verifier's law (본문에서는 "verifier's rule") — 법칙 문장과 채점이 쉬운 일의 다섯 조건: <https://www.jasonwei.net/blog/asymmetry-of-verification-and-verifiers-law> (2025-07-15)
- Andrej Karpathy, Verifiability — 소프트웨어 1.0과 2.0의 자동화 대상: <https://karpathy.bearblog.dev/verifiability/> (2025-11-17)
- Andrej Karpathy, Sequoia Ascent 2026 summary — 들쭉날쭉한 지능, 검증 가능성, 미학 보상: <https://karpathy.bearblog.dev/sequoia-ascent-2026/> (2026-04-30) · 원본 영상: <https://youtu.be/96jN2OCOfLs> (2026-04-29)
- Zhao 외, SpecBench: Measuring Reward Hacking in Long-Horizon Coding Agents: <https://arxiv.org/abs/2605.21384> (2026-05, 동료 심사 전)
- Wu, Tang, From Rebound to Remedy: <https://arxiv.org/abs/2604.01476> (2026-04, 동료 심사 전)
- Slocum 외, Diverse Preference Learning for Capabilities and Alignment: <https://arxiv.org/abs/2511.08594> (ICLR 2025)
- Zhang 외, Verbalized Sampling — 평가자의 익숙함 편향: <https://arxiv.org/abs/2510.01171> (2025-10)
- Jiang 외, Artificial Hivemind — 질의 2만 6천 개, 모델 간 동질성: <https://arxiv.org/abs/2510.22954> (NeurIPS 2025)
- Gunjal 외, Rubrics as Rewards: <https://arxiv.org/abs/2507.17746> (2025-07, 프리프린트)

### 12장: 과소명세

- Wu, Fard, HumanEvalComm: <https://arxiv.org/abs/2406.00215> (ACM TOSEM 2025)
- Vijayvargiya 외, Ambig-SWE (Interactive Agents to Overcome Ambiguity in Software Engineering): <https://arxiv.org/abs/2502.13069> (ICLR 2026, arXiv v3 2026-02)
- Zhang, Knox, Choi, Modeling Future Conversation Turns to Teach LLMs to Ask Clarifying Questions: <https://arxiv.org/abs/2410.13788> (ICLR 2025)
- Claude Code 문서, Best practices — 의도는 추론하지만 마음을 읽지는 못한다, 인터뷰 프롬프트: <https://code.claude.com/docs/en/best-practices> (2026-10)
- mattpocock/skills — `grill-me`, `grill-with-docs`의 인터뷰 규칙: <https://github.com/mattpocock/skills> (2026-10-12 확인, 인터뷰 규칙 파일 최종 수정 2026-10-07)

### 13~15장: 설계와 개발자

- Amazon, 2015 Letter to Shareholders — 양방향 문과 일방통행 문: <https://www.sec.gov/Archives/edgar/data/1018724/000119312516530910/d168744dex991.htm> (2016-04)
- Claude Code 문서, Security — 승인 피로, 네트워크 명령의 기본 동작: <https://code.claude.com/docs/en/security> (2026-10)
- Faros AI, The AI Productivity Paradox Report 2025 — PR 병합 98% 증가, 리뷰 시간 91% 증가, "핵심 병목은 사람의 승인": <https://www.faros.ai/blog/ai-software-engineering> (2025-07-23)
- Faros AI, The AI Engineering Report 2026: The Acceleration Whiplash — 개발자 약 2만 2천 명·4,000개 팀, 중간 리뷰 시간 5배, PR당 버그 +28%, 주당 배포 −11.7%: <https://www.faros.ai/research/ai-acceleration-whiplash> (2026년 2분기)
- Matt Pocock, Fixing the PR Bottleneck (발표 영상): <https://www.youtube.com/watch?v=LlgiOCmFG_w> · 스킬 원문: <https://github.com/mattpocock/skills> (2026-10-12 확인)
- Addy Osmani, Loop Engineering — 이해 부채와 편안한 자세: <https://addyosmani.com/blog/loop-engineering/> (2026-06-07)

### 부록

- Claude Code 문서, Keep Claude working toward a goal — `/goal`의 평가 방식과 종료, `/loop`·Stop 훅과의 비교: <https://code.claude.com/docs/en/goal> (2026-10)
- Claude Code 문서, Configure tool search — `ENABLE_TOOL_SEARCH`와 `alwaysLoad`: <https://code.claude.com/docs/en/mcp#configure-tool-search> (2026-10)
- Claude Code 문서, CLI reference — `--max-turns`, `--max-budget-usd`, `--permission-mode`: <https://code.claude.com/docs/en/cli-reference> (2026-10)
- Codex 설정 참조: <https://learn.chatgpt.com/docs/config-file/config-reference> (2026-10)
- MCP 블로그, 2026-07-28 스펙 발표 (stateless 전환): <https://blog.modelcontextprotocol.io/posts/2026-07-28/> (2026-07-28)
- ReAct (Yao 외, 2022): <https://arxiv.org/abs/2210.03629> · Reflexion (Shinn 외, 2023): <https://arxiv.org/abs/2303.11366>
- Anthropic, Building Effective Agents — evaluator-optimizer, orchestrator-workers: <https://www.anthropic.com/engineering/building-effective-agents> (2024-12)
