# 코딩 에이전트 도구 현황 조사 (2026년 10월 기준)

> 작성일: 2026-10-13
> 범위: ① 도구 종류와 선호도 순위, ② provider 수 대비 도구가 많아진 배경, ③ 도구별 강점, ④ 사용자 선택 기준, ⑤ Copilot·Cursor 점유율 하락 배경, ⑥ 오픈소스 에이전트가 선호되는 이유, ⑦ 오픈소스 에이전트 선호도 순위(Pi 포함), ⑧ Copilot 환경에서 다른 도구를 선택할 가능성에 대한 추론
> 각 항목 끝의 `[n]`은 문서 하단 **참고 자료** 번호입니다.

---

## 요약

- **사용률과 선호도 모두 1위는 Claude Code**입니다. 그 뒤를 GitHub Copilot, OpenAI Codex, Cursor가 따릅니다.
- 모델 provider는 몇 곳뿐인데 도구가 많아진 이유는 경쟁의 축이 **모델에서 하네스(Agent = Model + Harness)로 이동**했기 때문입니다.
- **Copilot과 Cursor의 하락**은 성능 문제라기보다 다음 세 가지가 겹친 결과입니다.
  - IDE 보조 도구에서 자율 에이전트로의 전환
  - 요금제 변경(종량제, 한도 축소)
  - 경쟁사의 저가 구독
- **오픈소스 에이전트**는 벤더 정책 리스크를 피하는 출구로 선호됩니다. 2026년 Anthropic의 서드파티 하네스 차단 사태가 이 흐름을 가속했습니다.
- 독립 오픈소스 하네스 순위는 **OpenCode가 1위, Pi가 2위**입니다. Pi는 하반기 성장세가 가장 큰 프로젝트입니다.
- **Copilot을 쓰는 회사**에서 다른 도구로의 이동 동기는 도구의 결함이 아닙니다. Claude Code·Codex는 한 회사 모델에 최적화되어 있는데, 이 환경은 여러 회사 모델이 열려 있어 둘 사이가 어긋나는 데서 생깁니다. 이동은 전면 교체가 아니라 **파워유저부터 "주력 + 보조" 형태로 번지는 부분 이동**이 될 가능성이 높습니다(8장).

---

## 1. 코딩 에이전트 도구 선호도 순위

### 1.1 주요 설문 수치

| 순위 | 도구 | JetBrains 업무 사용률 (2026년 5~7월) | Stack Overflow 2026 사용률 | "가장 좋아하는 도구" (Pragmatic Engineer) |
|---|---|---|---|---|
| 1 | Claude Code | 39% (1월 18%) | 65.5% | 46% |
| 2 | GitHub Copilot | 21% (1년 전 29%) | 58.7% | 9% |
| 3 | OpenAI Codex | 16% (1월 약 3%) | 29.5% | – |
| 4 | Cursor | 12% (1월 18%) | 25.5% | 19% |
| 5군 | OpenCode, Antigravity, Kilo/Cline, Junie, Windsurf, Amp, Augment, Factory, Zed | 각 10% 안팎 또는 그 이하 | | |

- **JetBrains 2026 Developer Ecosystem Survey** (전문 개발자 15,000명 이상, 2026년 5~7월) [1][2]
  - 90%가 코딩 에이전트를 매주 1회 이상, 68%가 매일 사용합니다.
  - Claude Code의 미국 내 사용률은 47%입니다.
- **Stack Overflow 2026 Developer Survey** (유효 응답 30,903건, 169개국) [3]
  - 코딩 에이전트·어시스턴트 문항은 12,255명이 응답했습니다.
  - 사용자의 73%가 매일 사용합니다.
- **Pragmatic Engineer 설문** (응답 906명, 2026년 1~2월) [4]
  - 응답자의 70%가 2~4개 도구를 함께 씁니다.
  - OpenCode, Gemini CLI, Antigravity는 각각 약 10%가 사용합니다.

### 1.2 최근 지형 변화

- **Google**: 2026년 6월 18일 Gemini CLI와 Gemini Code Assist IDE 확장을 종료하고 Antigravity CLI로 이전했습니다. 새 CLI는 오픈소스가 아니어서 커뮤니티의 반발이 있었습니다. [5][6]
- **Roo Code**: 2026년 5월 15일 저장소가 아카이브되었습니다. [7]
- **Cursor**: SpaceX가 600억 달러에 인수했습니다(8월 완료). 이에 따라 OpenAI는 **2026년 11월 12일부로 Cursor에 대한 모델 공급을 중단**합니다. Cursor 측은 OpenAI 모델 비중이 트래픽의 약 5%라고 밝혔습니다. [8][9][10]
- **Windsurf**: Google이 원래 팀을 인재 영입 방식으로 데려갔고, 제품 자체는 Cognition이 운영합니다. [4]

---

## 2. provider 수 대비 도구가 많아진 배경

### 2.1 핵심 개념: Agent = Model + Harness

하네스는 모델 바깥의 모든 것을 말합니다. 루프, 도구, 컨텍스트 선택, 권한, 서브에이전트, 메모리가 여기에 들어갑니다. 하네스는 모델에게 어떤 파일·문서·커밋·도구 출력을 보여줄지 결정합니다. [11][12]

```python
# 하네스의 본질(단순화): 모델은 교체 가능한 부품
class Harness:
    def __init__(self, model):          # Claude, GPT, Gemini, Qwen, 로컬 모델...
        self.model = model
        self.tools = {"read": read_file, "edit": edit_file, "bash": run_sandboxed}

    def run(self, task):
        ctx = self.select_context(task)          # 차별화 1: 컨텍스트 엔지니어링
        while True:
            action = self.model.next_action(ctx)
            if action.type == "done":
                return action.result
            if not self.permit(action):          # 차별화 2: 권한·안전장치
                continue
            obs = self.tools[action.tool](**action.args)
            ctx = self.compact(ctx + [obs])      # 차별화 3: 압축·재시도

Harness(model=AnthropicModel("opus")).run("버그 수정")
Harness(model=LocalModel("qwen-coder")).run("버그 수정")
```

### 2.2 배경 요인

1. **모델 성능이 수렴하고, 하네스가 성능을 좌우합니다.**
   - LangChain 팀은 모델을 바꾸지 않고 하네스 최적화만으로 Terminal Bench 2.0 순위를 30위에서 5위로 올렸습니다. [12]
   - Cursor는 모델이 범용재가 되고 승부는 하네스에서 난다고 봅니다. [11]
2. **진입장벽이 낮아지고 포크 생태계가 생겼습니다.**
   - Cline → Roo Code → Kilo Code로 이어지는 포크 계보가 있습니다.
   - Kilo CLI 1.0은 MIT 라이선스인 OpenCode 코드베이스 위에 만들어졌습니다. [13][14]
3. **비용 절감 수요가 큽니다.**
   - Sapiom은 OpenCode로 바꿔 AI 비용을 25% 줄일 것으로 예상합니다.
   - 토큰 단가는 내려가지만, 에이전트의 토큰 소모량이 커서 기업 청구액은 오히려 늘고 있습니다. [15]
4. **벤더 종속을 피하려고 사내 하네스를 구축합니다.**
   - Coinbase, Shopify, Ramp 등은 모델 비용을 내면서도 자체 루프를 만들어 모델을 교체 가능한 입력으로 둡니다. [16]
   - Cursor와 OpenAI 사례는 IDE 안의 모델 메뉴가 사실상 공급망이라는 점을 보여줬습니다. [10]

---

## 3. 도구별 강점 어필 포인트

| 도구 | 강점 |
|---|---|
| Claude Code | 복잡한 문제에서의 추론 깊이. Agent Teams(다중 에이전트), MCP, hooks [17] |
| Codex | CLI·IDE·ChatGPT 앱·웹을 아우르는 다중 접점, 격리된 클라우드 샌드박스 [18][19] |
| GitHub Copilot | 가장 넓은 IDE 지원과 기업 조달 강점. 2026년 2월 Claude·Codex 백엔드를 추가해 멀티모델화 [19] |
| Cursor | IDE 네이티브 경험. 모델에 구애받지 않는 SDK로 자사 하네스를 외부에 개방 [11] |
| Antigravity | 데스크톱과 CLI가 같은 하네스를 공유. 병렬 서브에이전트, 비동기 작업 [20] |
| OpenCode / Kilo / Cline | 오픈소스, BYOK, 로컬 모델. Kilo는 500개 이상 모델을 수수료 없이 지원. Cline은 SSO·감사 로그·VPC·에어갭 배포 제공 [7][21] |

---

## 4. 사용자의 도구 선택 기준

1. **회사 조달 정책이 개인 취향보다 우선합니다.** 직원 1만 명 이상 기업은 Copilot(56%)을, 소규모 스타트업은 Claude Code(75%)와 Cursor(42%)를 주로 씁니다. [4]
2. **비용과 사용량 한도.** 비용이 가장 큰 불만이고, 월 200달러 Max 요금제에서도 한도에 대한 불만이 나옵니다. [17]
3. **검증 가능성과 신뢰.** 약 절반은 검증할 수 있을 때만 AI 출력을 신뢰하고, 78.7%는 출처 표시가 중요하다고 답했습니다. [22]
4. **작업 유형별로 도구를 병행합니다.** 70%가 2~4개 도구를 함께 쓰며, 흔한 구성은 터미널의 Claude Code와 검토용 IDE를 나란히 띄우는 방식입니다. [4]
5. **기본값을 그대로 쓰는 층이 있습니다.** 약 8명 중 1명은 회사가 정한 기본 모델을 그대로 씁니다. [4]

---

## 5. GitHub Copilot·Cursor 점유율 하락의 배경

### 5.1 수치

- Copilot: 업무 사용률이 29%에서 21%로 내려갔습니다(JetBrains, 1년 대비). [1]
- Cursor: 1월 18%에서 5~7월 12%로 내려갔습니다. [1]

### 5.2 공통 요인: IDE 보조에서 자율 에이전트로의 전환

- Claude Code와 Codex처럼 IDE 바깥에서도 동작하는 에이전트가 성장하는 동안 Cursor는 줄었습니다. [2]

### 5.3 Cursor 고유 요인

- **2025년 요금제 사태로 신뢰를 잃었습니다.**
  - 고정 요청 한도를 "무제한 + 동적 속도 제한"으로 바꾼 뒤 품질 저하와 한도 조기 소진 불만이 나왔습니다. [23]
  - 대량 해지가 이어졌고, Cursor는 영향받은 사용자에게 전액 환불했습니다. [24]
- **가격 경쟁력이 약합니다.** 범용재화되는 시장에서 프리미엄 가격을 받고 있고, Claude Code로 옮긴 개발자는 성능보다 비용을 주된 이유로 꼽습니다. [25]
- **기업 고객은 남고 개인이 이탈합니다.** 전환 비용 때문에 기업은 남지만, 초기 유행을 만든 개인 개발자는 떠나고 있습니다. [25]
- **모델 공급처가 경쟁자입니다.** OpenAI의 공급 중단(11월 12일)이 대표적입니다. [8][10]

### 5.4 Copilot 고유 요인

- **종량제 전환.** 2026년 6월 1일 모든 요금제가 토큰 사용량 기반의 AI Credits(1크레딧 = 0.01달러)로 바뀌었습니다. [26]
- **한도 도입이 불투명했습니다.** 도입 초기에 사용량을 확인할 방법이 없었고, 많은 사용자가 한도에 자주 걸립니다. [23]
- **대기업 구매 의존 구조.** 기업 계약은 남아도 개발자의 실제 사용은 다른 도구로 옮겨갑니다. [4]

---

## 6. 오픈소스 에이전트 도구가 선호되는 이유

여기서는 자체 AI 요금제를 판매하는 회사의 에이전트는 제외합니다.

1. **벤더 정책 리스크에 대한 헤지 (결정적 계기)**
   - 2026년 1월 9일, Anthropic은 서드파티 도구가 Claude Pro/Max 구독으로 OAuth 인증하는 것을 차단했습니다.
   - 4월 4일에는 이를 모든 서드파티 하네스로 확대하고 종량제로 전환시켰습니다. [27]
   - 반면 OpenAI는 2026년 2월경 OpenCode, OpenHands, RooCode와 제휴해 Codex 구독을 서드파티에서도 쓸 수 있게 했습니다. [28]
   - 그 결과 OpenCode 사용자는 특정 벤더의 정책이 바뀌어도 "약간 불편해지는 정도"로 끝납니다. [27]
2. **모델 선택 자유와 비용 최적화**
   - 75개 이상의 provider를 쓸 수 있습니다.
   - 일상 작업은 저가 모델, 복잡한 작업은 프리미엄 모델로 나눌 수 있고, 로컬 모델로 오프라인 실행도 가능합니다. [29]
3. **보안과 폐쇄망 대응**
   - 로컬 모델을 쓰면 에어갭 환경에서 실행할 수 있고 코드가 외부로 나가지 않습니다. [30]
   - Cline은 VPC·에어갭 배포를 지원하는 엔터프라이즈 등급을 제공합니다. [7]
4. **투명성과 확장성.** 에이전트 루프를 읽고, 고치고, 포크할 수 있습니다. 반면 Claude Code는 내부 루프를 공개하지 않습니다. [31]
5. **커뮤니티 개발 속도.** OpenCode는 기여자 약 900명이 거의 매일 릴리스를 냅니다. [7]

사내 게이트웨이 연결 예시(개략):

```jsonc
// opencode.json
{
  "model": "company-gw/qwen-coder",
  "provider": {
    "company-gw": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "https://llm-gw.internal/v1" },
      "models": { "qwen-coder": {}, "claude-sonnet": {} }
    }
  }
}
// 벤더 정책이 바뀌어도 baseURL과 model만 교체하면 됨
```

**주의할 점**

- 기본 권한 범위가 넓은 경우가 있습니다. [32]
- 오픈웨이트 로컬 모델의 실제 성능은 아직 들쭉날쭉합니다. [7]
- 스타 수가 항의성 스타로 부풀려진 경우가 있습니다. [29]
- 프로젝트 지속성 리스크가 있습니다(Roo Code 아카이브, Aider 정체). [7][33]

---

## 7. 오픈소스 에이전트 선호도 순위 (Pi 포함)

### 7.1 독립 오픈소스 하네스

| 순위 | 도구 | 스타 (10/13) | 8/29 대비 | 라이선스 | 특징 |
|---|---|---|---|---|---|
| 1 | OpenCode | 211.7k | +9.7k | MIT | 사실상 표준. 설문에서도 사용률 약 10%, Staff+ 엔지니어의 선호도가 Copilot과 비슷 [4][33][44] |
| 2 | **Pi** | 114.8k | +16.8k | MIT | 최소 코어, 토큰 효율, 확장성. 1.0 출시 [33][34][45] |
| 3 | OpenHands | 90.8k | +5.8k | MIT | 샌드박스 기반 자율 실행, CI 헤드리스 [33][46] |
| 4 | Cline | 67.6k | +0.6k | Apache-2.0 | IDE·CLI·SDK 통합 [33][47] |
| 5 | Goose | 55.2k | +1.2k | Apache-2.0 | MCP 기반 범용 에이전트, Linux Foundation 이관 [33][48] |
| 6 | Kilo Code | 27.6k | +0.6k | MIT | Roo Code 이탈자 흡수 [33][49] |

- 스타 수는 **2026년 10월 13일** 각 GitHub 저장소 페이지에서 다시 확인한 값입니다(소수점 첫째 자리 반올림 표기). "8/29 대비"는 Pinggy의 8월 29일 수치 [33]와의 차이입니다.
- 6주 사이 순증가 폭은 **Pi(+16.8k)가 OpenCode(+9.7k)보다 큽니다.** 하반기 성장세가 가장 크다는 판단과 일치합니다.

### 7.2 모델 회사가 만든 오픈소스 CLI (참고, 순위 제외)

| 도구 | 스타 (10/13) | 8/29 대비 | 비고 |
|---|---|---|---|
| DeepSeek Harness | 243.1k | +40.1k | 8월 13일 공개, 알파 버전. 오픈소스 코딩 에이전트 중 스타 1위 [33][50] |
| Codex CLI | 128.1k | +9.1k | [33][51] |
| Qwen Code | 27.3k | +0.3k | [33][52] |
| Grok Build | 19.6k | −6.4k (확인 필요) | [33][53] |

- Codex CLI·Qwen Code·Grok Build 페이지는 조회 결과가 캐시 스냅샷일 수 있다는 표시가 있었습니다.
- Grok Build는 8월 수치보다 낮게 조회되어, 출처 간 집계 차이이거나 캐시 문제일 수 있습니다. 인용 전에 저장소에서 직접 확인을 권합니다.

### 7.3 Pi 상세

- **성장세**: 5월 54k → 8월 약 98k → 10월 초 약 113k → 10월 13일 114.8k(포크 14.6k). 기여자 296명, v1.0.4(10월 5일) [33][34][45]
- **설계 철학**
  - 시스템 프롬프트가 1,000토큰 미만입니다. Cline·OpenCode·Claude Code는 7,000~10,000토큰 수준입니다.
  - "lazy skills" 방식으로, 평소에는 한 줄 설명만 두고 호출될 때 전체를 불러옵니다. [33]
- **최소 코어**
  - MCP, 서브에이전트, plan mode, 할 일 목록, 권한 팝업 없이 출발했습니다. [35]
  - 기본 도구는 read, write, edit, bash 네 가지입니다. [36]
- **생태계**
  - 서드파티 패키지가 2,143개 있습니다(5월 기준). [36]
  - 9월 29일 v0.99.0에서 MCP를 추가했고, 10월 1일 Pi 1.0이 나왔습니다. [34]
- **배경**
  - OpenClaw(약 39만 스타)가 Pi의 구성요소 위에 만들어졌습니다.
  - 2026년 4월 8일 Earendil(Armin Ronacher)이 인수했습니다. [34]
- **약점**: 신생이라 실전 검증이 적고, 기본 기능이 적으며, 포크와 미러 저장소가 난립합니다. [33]

```python
# lazy skills 개념도
# 일반 하네스: 모든 도구 스키마를 매 턴 포함
system_prompt = BASE + "\n".join(t.full_schema for t in all_tools)

# Pi 방식: 한 줄 요약만 상주시키고, 호출 시 상세 내용을 로드
system_prompt = BASE + "\n".join(f"- {s.name}: {s.one_liner}" for s in skills)

def on_skill_call(name):
    skill = load_skill(name)       # 이 시점에 SKILL.md 본문과 스키마를 처음 읽음
    ctx.append(skill.full_instructions)
```

### 7.4 순위 해석 시 주의할 점

- **스타는 관심도이지 실사용이 아닙니다.** claw-code는 약 195k 스타지만 README 스스로 실사용을 권하지 않습니다. [33]
- **활동성을 함께 확인해야 합니다.** Aider는 5월 22일 이후 커밋이 없고, Trae Agent는 2월 이후 멈춰 있으며, Roo Code는 아카이브되었습니다. [33]
- **목적에 따른 선택**: 안정적인 기본 선택은 OpenCode, 하네스를 이해하고 사내용으로 개조하려면 Pi가 적합합니다.

---

## 8. Copilot 환경에서 다른 도구를 선택할 가능성에 대한 추론

**전제 환경**: GitHub Copilot Business를 쓰는 회사에서 Claude Code·Codex의 base URL을 `api.githubcopilot.com`으로 지정해 사용하는 경우입니다.

### 8.1 이 환경이 가진 구조적 특성

- **모델 선택권이 도구가 아니라 구독에 있습니다.**
  - Copilot 구독 하나로 Claude, GPT, Gemini 계열을 모두 쓸 수 있습니다.
  - 반면 Claude Code는 Anthropic 메시지 형식에, Codex는 OpenAI 형식에 맞춰 설계되어 있습니다.
  - 결국 모델 메뉴는 넓은데, 도구는 그중 한쪽만 제대로 활용하는 상황입니다.
- **하네스가 경험을 좌우합니다.** 모델 변경 없이 하네스 최적화만으로 Terminal Bench 순위가 30위에서 5위로 오른 사례가 있습니다. [12]
- **여러 도구 병행이 이미 일상입니다.** 개발자의 70%가 2~4개 도구를 함께 쓰므로, 도구를 하나 더 들이는 장벽이 낮습니다. [4]
- **토큰 과금으로 하네스 효율이 곧 비용이 되었습니다.** 2026년 6월 1일부터 Copilot은 입력·출력·캐시 토큰을 모델별 API 단가로 계산해 AI Credits를 차감합니다. [37][38]

### 8.2 이동을 끌어당기는 힘 ① 여러 회사 모델의 조합

개발자가 원하는 것은 "최고의 모델 하나"가 아니라 **작업 단계마다 맞는 모델**입니다.

```python
# 개발자가 머릿속으로 이미 하고 있는 라우팅을 하네스가 직접 지원하는 형태
roles = {
    "plan":    "github-copilot/claude-opus",     # 설계·분해
    "code":    "github-copilot/claude-sonnet",   # 구현
    "review":  "github-copilot/gpt-5",           # 다른 회사 모델로 교차 검토
    "explore": "onprem/qwen-coder",              # 대량 탐색·요약 (사내 모델)
}
```

- **교차 검토의 가치**
  - 같은 회사 모델끼리 검토하면 비슷한 맹점을 공유합니다.
  - 다른 회사 모델로 검토하면 품질상의 이득이 분명하지만, 한 회사 전용 도구에서는 이 작업이 번거롭습니다.
- **막혔을 때의 탈출구**
  - 한 모델이 같은 실수를 반복할 때, 세션을 유지한 채 모델만 바꾸는 경험은 강하게 각인됩니다.
  - OpenCode는 세션 중 provider를 바꿀 수 있고, Crush는 세션 중 모델 교체를 대표 기능으로 내세웁니다. [33][42]
- **모델 출시 주기와의 정합성**
  - 몇 달마다 새 모델이 나오고 회사별 1위가 번갈아 바뀝니다.
  - 도구를 갈아타지 않고 모델 줄 하나만 바꾸면 되므로, 손에 익은 환경을 유지할 수 있습니다.
- **Copilot과 사내 온프레미스 모델의 혼합**
  - 사내 OpenAI 호환 게이트웨이(LiteLLM 등)를 Copilot과 함께 provider로 등록할 수 있습니다.
  - 민감하거나 대량인 작업은 사내 모델로, 어려운 추론은 Copilot 모델로 나눌 수 있습니다.

```jsonc
// opencode.json (개략 예시)
{
  "model": "github-copilot/claude-sonnet",
  "small_model": "onprem/kimi",
  "provider": {
    "onprem": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://litellm.internal:4000/v1" },
      "models": { "kimi": {} }
    }
  }
}
```

### 8.3 이동을 끌어당기는 힘 ② 개인화된 하네스 튜닝의 자유

- **관찰하고 바로 고치는 순환**
  - 시스템 프롬프트, 도구 정의, 권한 모델이 저장소에 공개되어 있어, 에이전트가 이상하게 행동하면 원인을 보고 직접 고칠 수 있습니다. [33]
  - 닫힌 도구에서는 "프롬프트를 바꿔 다시 시도"하는 것밖에 할 수 없습니다. 이 차이는 사용 기간이 길수록 커집니다.
- **비워 둔 코어 위에 자기 도구를 쌓기**
  - Pi는 MCP, 서브에이전트, plan mode, 권한 팝업 없이 출발했습니다.
  - 필요한 기능은 npm이나 git에서 확장으로 설치하는, 이른바 맞춤복 모델입니다. [35]
  - 원작자는 이를 "가장 조종하기 쉬운 하네스"라고 표현합니다. [35]
- **대화를 나무처럼 갈라서 탐색하기**
  - Pi의 `/tree`는 세션 분기 탐색, 필터링, 북마크, HTML·gist 내보내기를 지원합니다. [43]
  - 실패한 시도를 컨텍스트에서 떼어내고 깨끗한 지점에서 다시 시작할 수 있습니다.
- **되돌리기에 대한 안전감**: CodeWhale은 작업 공간 스냅샷, undo/restore, 세 가지 승인 모드를 제공합니다. [33]
- **조직 맥락을 루프에 직접 넣기**
  - 사내 빌드 시스템(Yocto 등), 코딩 규칙, 테스트 절차를 하네스의 도구와 루프 안에 넣으면 에이전트가 "우리 팀 방식"으로 일합니다.
  - Pi는 interactive, print/JSON, RPC, SDK의 네 가지 실행 모드를 제공해 사내 자동화(PR 리뷰 봇 등)의 엔진으로도 쓸 수 있습니다. [36]
- **토큰 효율 튜닝**
  - 하네스 고정 오버헤드는 에이전트 스텝 수만큼 곱해집니다.
  - Pi는 시스템 프롬프트가 1,000토큰 미만이고, 주요 하네스는 약 7,000~10,000토큰입니다. [33]

```python
# 하네스 고정 오버헤드가 세션 비용에 미치는 영향 (개념 계산)
steps, cache_ratio = 60, 0.1
overhead = lambda sp: sp + sp * (steps - 1) * cache_ratio
print(overhead(8000), overhead(1000))   # 약 55,200 vs 약 6,900 (정가 환산 토큰)
```

### 8.4 이동을 막는 반대 힘

- **모델별 튜닝 격차**
  - Cursor는 모델마다 몇 주씩 하네스를 조정합니다. [11]
  - 범용 하네스는 같은 모델을 써도 전용 도구보다 손에 덜 붙을 수 있습니다. 모델 선택은 자유롭지만, 모델마다 조금씩 덜 똑똑하게 느껴지는 트레이드오프입니다.
- **튜닝은 자유이자 비용입니다.** 다듬는 시간이 작업 시간을 잡아먹고, 다수 개발자는 잘 만들어진 기본값을 선호합니다.
- **기본 설정과 기능 공백**
  - OpenCode는 에이전트 권한이 기본적으로 넓다는 지적이 있습니다. [32]
  - Pi는 기본 제공 기능이 적어 초기 조합 비용이 듭니다. [33]
- **네트워크 효과.** 팁, 스킬, 트러블슈팅 공유에는 다수가 쓰는 도구가 유리합니다.
- **성숙도와 지속성.** Roo Code 아카이브, Aider 정체처럼, 공들여 길들인 도구가 사라질 위험이 있습니다. [33]
- **공식 지원 경로의 차이**
  - GitHub는 OpenCode와 정식 제휴해 Copilot Business 구독으로 로그인할 수 있게 했습니다. [39]
  - GitHub의 에이전트 워크플로(gh-aw)도 Pi, OpenCode, Crush를 github provider로 기본 연결합니다. [40]
  - 반면 Claude Code를 Copilot 엔드포인트로 쓰는 구성의 공식 지원 범위나, Anthropic 서버 측 기능에 의존하는 기능의 동작 여부는 별도 확인이 필요합니다.

### 8.5 종합 판단

| 개발자 유형 | 이동 가능성 | 예상 형태 |
|---|---|---|
| 시니어·Staff+ (자기 방식이 확고함) | 높음 | Pi 같은 최소 하네스를 개인화해 주력으로 사용 |
| 여러 모델을 비교·교차 검토하는 사람 | 중간~높음 | Claude Code 주력 + OpenCode 보조 |
| 일반 개발자 (기본값 선호) | 낮음 | Claude Code 유지, 동료의 공유 설정이 생기면 따라감 |

- **근거 데이터**: OpenCode는 전체 사용량이 Copilot의 4분의 1 수준인데도, "가장 좋아하는 도구" 언급 수는 Copilot과 비슷했습니다. Staff+ 엔지니어층에서는 동률이었습니다. 이는 **소수 숙련자가 먼저 깊게 쓰는 패턴**을 보여줍니다. [4]
- **예상 확산 경로**: 파워유저가 개인화된 하네스로 먼저 이동하고, 그들의 설정과 확장이 팀에 공유되고, 표준 설정이 생기면 일반 개발자가 따라옵니다.
- **결정적 변수**: 도구 자체의 우열보다 **사내에서 누가 좋은 하네스 설정을 먼저 만들어 공유하느냐**입니다.
- **실무 권고**: 결정 전에 같은 작업 세트를 두 도구로 돌려 **크레딧 소모와 결과 품질을 함께 비교**하십시오.

---

## 참고 자료

1. ADTmag – JetBrains Survey Finds AI Coding Agents Becoming Routine Developer Tools (2026-08-27): https://adtmag.com/articles/jetbrains-survey-finds-ai-coding-agents-becoming-routine-developer-tools.aspx
2. AI Understanding – 36Kr reports Claude Code led GitHub Copilot in JetBrains' 2026 developer survey: https://aiunderstanding.org/news/36kr-reports-claude-code-led-github-copilot-in-jetbrains-2026-developer-survey
3. Visual Studio Magazine – Stack Overflow 2026 Survey: VS Code Leads, Visual Studio Takes Third (2026-10-06): https://visualstudiomagazine.com/articles/2026/10/06/stack-overflow-2026-survey-vs-code-leads-visual-studio-takes-third.aspx
4. The Pragmatic Engineer – AI Tooling for Software Engineers in 2026: https://newsletter.pragmaticengineer.com/p/ai-tooling-2026
5. InventiveHQ – Gemini CLI Is Being Retired on June 18: https://inventivehq.com/blog/gemini-cli-deprecated-antigravity-cli-migration
6. Let's Data Science – Google replaces Gemini CLI with Antigravity CLI: https://letsdatascience.com/news/google-replaces-gemini-cli-with-antigravity-cli-574b2056
7. wetheflywheel – OpenCode vs Roo Code vs Cline: https://wetheflywheel.com/en/comparisons/opencode-vs-roo-code-vs-cline/
8. tbreak – OpenAI cuts off Cursor after SpaceX acquisition: https://tbreak.com/openai-cuts-off-cursor-spacex-acquisition/
9. Channel Insider – OpenAI to Cut Cursor Off From Its Models: https://www.channelinsider.com/ai/news-openai-cursor-model-access-spacex-acquisition/
10. JOPR – OpenAI will cut Cursor's access to its models: https://jopr.org/news/detail/openai-will-cut-cursors-access
11. The New Stack – Cursor's $60 billion bet is on the harness, not the model: https://thenewstack.io/cursor-sdk-harness/
12. Faros AI – Harness engineering: What makes AI coding agents work in 2026: https://www.faros.ai/blog/harness-engineering
13. Morph – Kilo Code vs OpenCode: https://www.morphllm.com/comparisons/kilo-code-vs-opencode
14. Morph – Kilo Code vs Cline: https://www.morphllm.com/comparisons/kilo-code-vs-cline
15. The Next Web – The AI coding fight just moved from the model to the harness: https://thenextweb.com/news/cursor-price-hikes-claude-code-harness-shift
16. SoftwareSeni – Build or Buy a Coding Agent Harness: https://www.softwareseni.com/build-or-buy-a-coding-agent-harness-and-what-ownership-really-costs/
17. Fungies – 7 Best AI Coding Agents for Developers in 2026: https://fungies.io/?p=37687
18. Fungies – Best AI Coding Agents 2026: https://fungies.io/?p=39179
19. Morph – 14 Best AI Coding Agents in 2026: https://www.morphllm.com/best-ai-coding-agents-2026
20. Toolhalla – Windsurf vs Google Antigravity: https://toolhalla.ai/compare/windsurf-vs-google-antigravity
21. AI Agent Index – Kilo Code: https://theaiagentindex.com/agents/kilo-code
22. IT Brief – Stack Overflow survey finds developers wary of AI tools: https://itbrief.co.uk/story/stack-overflow-survey-finds-developers-wary-of-ai-tools
23. heise – Price hike for AI coding tools: The Free Lunch Is Over: https://www.heise.de/en/background/Price-hike-for-AI-coding-tools-The-Free-Lunch-Is-Over-10511480.html?seite=all
24. Analytics India Magazine – Cursor's Pricing Backlash Sparks Developer Exodus: https://analyticsindiamag.com/ai-features/cursors-pricing-backlash-sparks-developer-exodus
25. UC Strategies – Cursor hit $2B revenue while developers quietly switched to cheaper rivals: https://ucstrategies.com/news/cursor-hit-2b-revenue-while-developers-quietly-switched-to-cheaper-rivals/
26. tech-insider – GitHub Copilot market share 2026: https://tech-insider.org/github-copilot-market-share-2026/
27. The New Stack – Why 157,000 developers are hedging against Anthropic with OpenCode: https://thenewstack.io/anthropic-claudecode-opencode-split/
28. falcao.org – Anthropic's Third-Party Claude Crackdown: https://falcao.org/posts/anthropic-claude-access-crackdown-ecosystem-fallout/
29. Daniel Vaughan – OpenCode vs Codex CLI: https://codex.danielvaughan.com/2026/04/09/opencode-vs-codex-cli/
30. imseankim blog – OpenCode: https://blog.imseankim.com/?p=11056
31. Tensorlake – OpenCode: The Best Claude Code Alternative: https://www.tensorlake.ai/blog/opencode-the-best-claude-code-alternative
32. DEV Community – OpenCode: The Open Source Coding Agent That Doesn't Lock You In: https://dev.to/playfulprogramming/opencode-the-open-source-coding-agent-that-doesnt-lock-you-in-pn3
33. Pinggy – Best Open Source CLI Coding Agents in 2026 (2026-08-28 업데이트): https://pinggy.io/blog/best_open_source_cli_coding_agents/
34. eesel – Pi coding agent explained (2026-10-07): https://www.eesel.ai/blog/pi-coding-agent
35. Pasquale Pillitteri – Pi Coding Agent: https://pasqualepillitteri.it/es/news/5651/pi-coding-agent-lo-que-no-construyeron
36. Implicator – Pi is not a Claude Code rival, it is a harness rebellion: https://www.implicator.ai/pi-is-not-a-claude-code-rival-it-is-a-harness-rebellion/
37. heise – GitHub removes free models from Copilot plans (토큰 과금 전환): https://heise.de/-11275252
38. The Decoder – GitHub Copilot switches to token-based billing in June 2026: https://the-decoder.com/github-copilot-switches-to-token-based-billing-in-june-2026/
39. GitHub Changelog – GitHub Copilot now supports OpenCode (2026-01-16): https://github.blog/changelog/2026-01-16-github-copilot-now-supports-opencode/
40. GitHub gh-aw PR #31758 – Pi/OpenCode/Crush provider metadata: https://github.com/github/gh-aw/pull/31758
41. Didier Durand – OpenCode, the OSS coding agent: https://didierdurand.substack.com/p/opencode-the-oss-coding-agent-80000
42. DevToolLab – Best Open Source Alternatives to Claude Code in 2026: https://devtoollab.com/blog/open-source-alternatives-claude-code
43. Pasquale Pillitteri – Pi Coding Agent (PT, /tree 세션 분기): https://pasqualepillitteri.it/pt/news/5656/pi-coding-agent-o-que-nao-construiram
44. GitHub – anomalyco/opencode (2026-10-13 조회): https://github.com/anomalyco/opencode
45. GitHub – earendil-works/pi (2026-10-13 조회): https://github.com/earendil-works/pi
46. GitHub – OpenHands/OpenHands (2026-10-13 조회): https://github.com/OpenHands/OpenHands
47. GitHub – cline/cline (2026-10-13 조회): https://github.com/cline/cline
48. GitHub – aaif-goose/goose (2026-10-13 조회): https://github.com/aaif-goose/goose
49. GitHub – Kilo-Org/kilocode (2026-10-13 조회): https://github.com/Kilo-Org/kilocode
50. GitHub – deepseek-ai/deepseek-harness (2026-10-13 조회): https://github.com/deepseek-ai/deepseek-harness
51. GitHub – openai/codex (2026-10-13 조회): https://github.com/openai/codex
52. GitHub – QwenLM/qwen-code (2026-10-13 조회): https://github.com/QwenLM/qwen-code
53. GitHub – xai-org/grok-build (2026-10-13 조회): https://github.com/xai-org/grok-build

### 추가 참고 (본문 미인용 보조 자료)

- Stack Overflow Blog – The results of the 2026 Developer Survey: https://stackoverflow.blog/2026/10/06/the-results-of-the-2026-developer-survey-are-here/
- VentureBeat – Anthropic cracks down on unauthorized Claude usage by third-party harnesses: https://venturebeat.com/ai/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses/
- The New Stack – Anthropic's harness shakeup "just fragments workflows": https://thenewstack.io/anthropic-claude-harness-restrictions/
- Parallel – OpenCode vs Claude Code: https://parallel.ai/ai/articles/opencode-vs-claude-code-a-2026-comparison-for-developers
