---
name: gcp-gmail
description: GCP에서 보내는 Gmail 알림(`gcp-no-reply` 류) 라벨 메일을 읽어 메일 1건당 Slack 부모 메시지를 만들고, 본문 요약·영향받는 프로젝트·To Action을 그 메시지의 스레드 답글로 발송합니다. 라벨/채널은 `config.yaml`, 중복 방지는 `state.json`. 사용자가 "gcp 메일 정리", "/gcp-gmail", "메일 다이제스트" 같은 표현을 쓰면 트리거됩니다.
---

# GCP Gmail Digest Skill

GCP가 발송하는 Gmail 알림(`gcp-no-reply`, `gcp-no-reply-actionrequired` 등)을 모아 Slack 채널에 정리해 보냅니다.

**핵심 형식**: 메일 1건당 Slack 부모 메시지 1개 + 그 메시지의 **스레드(thread)에 요약·영향 프로젝트·To Action 답글**. 다이제스트를 한 묶음 메시지로 합치지 않습니다.

## 동작 개요

1. `~/.claude/skills/gcp-gmail/config.yaml` 로드
2. `~/.claude/skills/gcp-gmail/state.json` 로드 (이미 처리한 thread ID, 마지막 실행 시각)
3. 각 라벨에 대해 Gmail 검색 → 신규 thread만 본문 추출
4. 각 thread마다:
   - **부모 메시지** 발송: `[제목] - 날짜` 헤더 + 발신자 + Gmail 링크
   - **스레드 답글** 발송: 요약 + 영향받는 프로젝트 + To Action
5. 처리한 thread ID를 `state.json`에 기록

## 환경

- Gmail MCP: `mcp__claude_ai_Gmail__*`
- Slack MCP: `mcp__claude_ai_Slack__*`

## 설정 파일

### config.yaml
- `labels`: 다이제스트 대상 라벨 목록 (예: `["gcp-no-reply", "gcp-no-reply-actionrequired"]`)
- `slack.channel`: 발송 대상 채널 ID 또는 이름 (예: `"C0AQV00LHSB"`)
- `lookback_hours`: 최근 N시간 이내 메일만
- `max_threads_per_run`: 한 번 실행에서 처리할 최대 thread 수
- `summary.language`, `summary.max_summary_lines`: 요약 옵션

`labels`가 비어 있거나 `slack.channel`이 비어 있으면 skill은 즉시 실행을 거부합니다.

### state.json
- `last_run_at`: ISO8601
- `processed_thread_ids`: 처리한 thread ID 배열. 무한정 커지지 않게 **최근 1000개**까지만 유지.

## 실행 절차

### 1. Config / State 로드 & 검증

```
Read ~/.claude/skills/gcp-gmail/config.yaml
Read ~/.claude/skills/gcp-gmail/state.json
```

- `slack.channel` / `labels` 검증 → 비어 있으면 안내 후 즉시 종료
- `state.json`이 없으면 빈 구조로 생성

### 2. 라벨별 thread 조회 (last_run_at 기반)

신규 메일만 효율적으로 가져오기 위해 `state.last_run_at` 기준으로 좁힙니다. **24시간 sliding window(`newer_than:24h`)는 사용하지 않습니다** — 사용자가 직관적으로 느끼는 "신규"와 어긋나고, cron 주기와도 맞지 않음.

**기본 쿼리 (state.last_run_at이 있을 때)**:

`last_run_at`(ISO8601, KST)에서 **날짜 부분만** 뽑아 Gmail 검색의 `after:` 연산자에 넣습니다.

```
mcp__claude_ai_Gmail__search_threads
  q: "label:<라벨명> after:<YYYY/M/D in KST>"
  pageSize: <max_threads_per_run>
```

Gmail의 `after:`는 날짜 단위라 같은 날 안의 더 이른 메일도 잡힙니다. 따라서 검색 결과의 각 `message.date`를 client-side에서 `> state.last_run_at`으로 한 번 더 필터링합니다.

**첫 실행 fallback** — `state.last_run_at`이 `null`이거나 7일 이상 지난 경우:

```
q: "label:<라벨명> after:<오늘 자정 KST 날짜 YYYY/M/D>"
```

이렇게 하면 "오늘 들어온 것" 직관과 일치하고, 누락 위험은 state-driven 흐름과 dedupe(`processed_thread_ids`)가 막아줍니다.

라벨에 공백·슬래시가 있어도 Gmail은 따옴표 없이 동작하지만, 안전을 위해 `label:"<이름>"` 형식으로 감싸도 됩니다.

### 3. 중복 필터 & 본문 추출

검색 결과의 각 `thread.id`에 대해:
- `state.processed_thread_ids`에 있으면 건너뜀
- 없으면 `mcp__claude_ai_Gmail__get_thread`로 전체 본문 가져오기

신규 thread가 **0건**이면 — Slack 발송 생략, 대화창에만 "처리할 신규 메일 없음" 보고.

### 4. 본문 분석 (GCP 알림 특화)

GCP no-reply 메일 본문에서 추출할 핵심 정보. 목표는 **"이 메일이 실제로 무엇을 알리고 무엇을 시키는지"** 를 한국어로 명료하게 옮기는 것. 단순히 카테고리만 분류하고 "권한 변경 필요" 식으로 모호하게 끝내지 않습니다.

**a. 한국어 제목 (필수)** — 영문 제목을 그대로 옮기지 않고 **한국어로 번역·압축** 합니다.
- 대괄호 태그(`[Action Required]`, `[Action Advised]`, `[Deprecation]` 등)는 한국어 의미로 변환 (`[조치 필요]`, `[권장 조치]`, `[Deprecation]`).
- 제품·고유명사(`Unified Rules`, `Cloud Storage`, `BigQuery` 등)와 권한/API 이름(`chronicle.dashboards.get` 등)은 **번역하지 말고 원문 유지**.
- 한 줄로 무엇이 변하는지 즉시 알 수 있게. 예시:
  - 원: `[Action Advised] Update custom IAM role permissions for Unified Rules Dashboard access`
  - 번: `[권장 조치] Unified Rules Dashboard 접근용 Custom IAM Role 권한 업데이트 필요`

**b. 알림 종류 식별** — 제목/본문 패턴으로 분류:
- *Deprecation / Sunset* — 서비스/API 종료 예정
- *Action required / Advised* — 사용자 조치 필요 (특히 `gcp-no-reply-actionrequired` 라벨)
- *Quota / Limit* — 쿼터·한도 관련
- *Billing* — 결제·청구
- *Security / IAM* — 보안 권고, 의심스러운 접근
- *Maintenance* — 예정된 유지보수
- *Informational (FYI)* — 정보성, 조치 불필요

**c. 본문 핵심 추출 (요약의 재료)** — 다음 항목을 본문에서 찾아 사실 그대로 옮깁니다:
- **무엇이 바뀌는가** — 변경되는 기능/제품/서비스 이름
- **언제부터** — 효력 발생 일자 (변환 후 `YYYY-MM-DD`)
- **왜** — 메일이 사유를 명시하면 한 줄로 (예: GA 전환, 보안 강화, 비용 정책 변경)
- **구체적 식별자** — 권한 이름(`secops.dashboards.viewer`), API 엔드포인트, 서비스 SKU, 정책 키 등. **원문 그대로** 보존. 단순히 "권한"이라고만 적지 않습니다.
- **현재 상태 vs 변경 후 상태** — 본문이 before/after를 명시하면 둘 다 적습니다.

**d. 영향받는 프로젝트** — 메일에 다음 표현 중 하나가 나오면 **그 다음에 오는 프로젝트 ID/번호 목록을 모두 추출**:
- "Your affected projects are listed below"
- "Affected projects:"
- "Project(s):"
- "The following projects are affected"

추출한 프로젝트 식별자는 원문 그대로(예: `hybe-prod-data-platform`, `123456789012`). 본문에 프로젝트가 명시되지 않으면 **이 섹션 자체를 생략**합니다 — "조직 전체", "수신자 대상" 같은 추정 문구로 채우지 않습니다.

**e. 마감일·중요 날짜** — "by YYYY-MM-DD", "deadline", "before", "as of", "starting on" 같은 표현 옆 날짜. 시간/타임존 제거하고 `YYYY-MM-DD`만 남깁니다.

**f. To Action (사용자가 실제로 해야 할 것)** — 메일 본문에 적힌 **기술적·운영적 조치**만 추출합니다:
- "You can/should/must …", "Please …", "We recommend …", "To resolve this …", "Add the following permission …" 같은 표현에서 구체적인 동사구 + 대상(권한명/리소스명).
- 가능하면 "{어떤 IAM Role}에 {어떤 권한}을 추가" 형태로 식별자까지 포함.
- **금지**: "사내 공유/보관 처리", "팀 내부 전달", "중복 수신본이라 대응 불필요" 같은 운영 절차는 메일 본문에 명시되어 있지 않으면 **자동 추가하지 않습니다**. 사용자가 직접 판단할 일.
- 모호한 인사말("문의 바랍니다")은 액션이 아닙니다.
- 정보성 메일이면 `• 없음 (정보성 알림)`.

**g. 답글에 포함하지 않는 것 (중요)**:
- 수신자/수신 대상 정보 (개인 메일 / 조직 Essential Contact / 팀 별칭 — 사용자가 도움받지 못함)
- 같은 공지가 다른 라벨/주소로 중복 수신되었다는 메타 정보 자체
- 메일 도입부 안내문구(`You are receiving this announcement because …`) — 원문 발췌 후보에서 제외
- 푸터(unsubscribe 링크, 회사 주소 등)

### 5. Slack 발송 — 메일별 부모 + 스레드 답글

각 thread마다 **반드시 다음 순서**로 호출합니다:

**Step A — 부모 메시지** (한국어 번역 제목 + 날짜만)
```
mcp__claude_ai_Slack__slack_send_message
  channel: <config.slack.channel>
  text: |
    📢 *<한국어 번역·압축한 제목>*
    🗓 <YYYY-MM-DD>
    🏷️ <라벨>
    🔗 <Gmail link: https://mail.google.com/mail/u/0/#inbox/<thread_id>>
```

규칙:
- 1번째 줄에 **한국어 제목**을 노출 (영문 원제는 부모 메시지에 적지 않음 — 클릭하면 원문이 보임)
- 날짜는 `YYYY-MM-DD`만. 시간·타임존(KST, UTC) 표기 금지
- 발신자/수신자 줄(`👤 …`) 추가하지 않음
- `gcp-no-reply-actionrequired` 라벨이면 첫 줄 이모지를 📢 대신 **🚨** 로 교체하고, 한국어 제목 앞에 `[조치 필요]` 가 없으면 강제로 추가

응답에서 `ts` 값을 받아 둡니다 (= 부모 메시지의 timestamp).

**Step B — 스레드 답글** (위에서 받은 `ts`를 `thread_ts`로 전달)
```
mcp__claude_ai_Slack__slack_send_message
  channel: <config.slack.channel>
  thread_ts: <부모 ts>
  text: |
    🔖 *알림 종류*: <분류된 카테고리 — 한 줄>

    📝 *무슨 내용인지*
    <본문 핵심 3~5줄. 무엇이 / 언제부터 / 왜 / 어떤 식별자가 관련되는지 구체적으로.
     "권한이 필요하다"가 아니라 "<역할>에 <권한명> 권한이 필요하다" 수준으로.>

    🎯 *영향받는 프로젝트*  ← 본문에 명시된 경우에만 섹션 자체를 출력
    • <project-id 1>
    • <project-id 2>

    ⏰ *마감/주요 날짜*  ← 메일에 명시된 날짜가 있을 때만
    • <YYYY-MM-DD>: <무엇이 일어나는지>

    ✅ *To Action*
    • <해야 할 일 1 — 동사 + 대상 식별자. 예: "secops_dashboard_viewer 역할에 chronicle.dashboards.get 권한 추가">
    • <해야 할 일 2>
    (조치 불필요한 정보성 메일이면 "• 없음 (정보성 알림)")

    📎 *원문에서 발췌*  ← 가장 핵심을 담은 1~2문장만. 도입부/안내문(`You are receiving …`) 인용 금지
    > <원문 인용>
```

답글 작성 시 추가 규칙:
- 수신자(개인 메일 / 조직 Essential Contact / 팀 별칭)에 대한 언급을 적지 않습니다.
- 같은 공지가 다른 라벨/주소로도 수신되었다는 사실 자체를 답글에 적지 않습니다 (사용자가 thread를 직접 비교).
- 본문에서 권한 이름·역할 이름·API 이름·SKU가 등장하면 한 번 이상 정확히 인용합니다.
- 한국어 작성. 단, 식별자·고유명사는 영문 그대로.

### 5.5. FinOps Agent mention (조건부)

부모 + 요약 답글까지 발송한 thread에 대해, 다음 **두 조건 중 하나라도** 충족하면 **같은 thread에 추가 답글로 `@gcp-finops-agent` 검토 요청**을 보냅니다.

**트리거 조건**

- **(A) 영향받는 프로젝트가 본문에 명시됨** — Step 4.d에서 추출한 프로젝트 ID 목록이 비어있지 않은 경우 ("Your affected projects are listed below", "Affected projects:", "Project(s):", "The following projects are affected" 다음에 실제 프로젝트 ID/번호가 명시된 경우).
- **(B) 가격·비용 관련 메일** — 다음 중 하나:
  - Step 4.b에서 분류된 카테고리가 `Billing` 또는 `Quota / Limit`
  - 본문 또는 제목에 다음 키워드 발견 (대소문자 무관):
    - en: `price`, `billing`, `cost`, `rate`, `increase`, `credit`, `free tier`, `invoice`, `SKU`, `fee`, `charge`, `discount`, `pricing`
    - ko: `가격`, `요금`, `청구`, `비용`, `단가`

두 조건이 다 충족되지 않으면 **이 단계 스킵**. mention 댓글은 잡음 방지를 위해 명확한 신호가 있을 때만 보냅니다.

단, 정보성(FYI) 메일이 키워드 우연 일치로만 매칭되면 호출하지 않음 — 본문이 실제 비용·청구 변경을 시사하는지 한 번 더 확인.

**호출**

부모 메시지의 `ts`(Step 5.A에서 받은 값)를 **다시** `thread_ts`로 전달해 같은 thread에 추가 답글로 발송. (요약 답글의 ts가 아니라 root 부모 ts를 사용 — Slack thread는 root 기준)

```
mcp__claude_ai_Slack__slack_send_message
  channel: <config.slack.channel>
  thread_ts: <Step 5.A의 부모 ts>
  text: |
    <@U0AP7558R70>

    ❓ *FinOps 검토 요청*
    • 트리거 사유: <"영향받는 프로젝트 명시" / "가격·비용 관련" / 둘 다 — 어떤 조건으로 매칭됐는지>
    • 메일 요지: <한 줄. 무엇이 / 언제부터 / 어떤 청구·비용 측면>
    • 영향받는 프로젝트: <목록>   ← 조건 (A)일 때
    • 감지된 키워드: <쉼표 구분>   ← 조건 (B)일 때

    확인 부탁드립니다:
    • 본 변경이 우리 조직 청구에 미치는 영향 추정
    • 영향 프로젝트의 현재 사용량·비용 추세 (확인 가능하면)
    • 권장 액션 (마이그레이션 / 최적화 / 모니터링 강화)
```

**mention 대상 정보**
- 봇 표시명: `gcp-finops-agent` (Slack username: `gcpfinopsagent`)
- Slack user ID: **`U0AP7558R70`** — mention syntax `<@U0AP7558R70>` 그대로 사용
- 검증된 채널: `#테스트-채널` (`C0AQV00LHSB`)
- **주의**: 평문 `@gcp-finops-agent` 또는 `@gcpfinopsagent`로 보내면 API 메시지에서 Slack이 username을 자동 변환하지 않아 **평문으로만 표시**되고 mention 알림이 가지 않습니다. 반드시 `<@USER_ID>` 형식 사용.

**가드레일**

- mention은 동일 thread당 **1건만** 발송. 같은 thread에 mention을 또 보내지 않음.
- mention 댓글 발송이 실패해도 해당 thread는 정상 처리된 것으로 간주하고 `state.json`에 thread.id를 기록합니다. mention만 재시도하지 않음 (다음 실행 때 같은 thread를 다시 안 보내야 하므로).
- 사용자 본인을 mention하거나, agent 외 다른 user/group을 mention하지 않음.

### 6. State 갱신

```json
{
  "last_run_at": "<현재 ISO8601>",
  "processed_thread_ids": [
    ...기존(최근 1000개 유지),
    ...이번에 부모+답글 발송에 성공한 thread.id
  ]
}
```

`Write ~/.claude/skills/gcp-gmail/state.json`

- 부모는 보냈지만 답글 발송이 실패한 thread는 **state에 기록하지 않습니다** (다음 실행 때 재처리되도록). 부모 발송 실패도 마찬가지.
- 단, 같은 thread에 두 번 부모가 생기는 걸 막기 위해, 답글 실패 시 사용자에게 "thread X는 답글 미발송 — 재실행 시 부모가 중복 생성될 수 있음"이라고 명시.

### 7. 사용자에게 결과 보고

```
GCP Gmail Digest 완료
- 대상 라벨: gcp-no-reply, gcp-no-reply-actionrequired
- 신규 thread: 5건 (스킵: 12건)
- Slack 발송: #gcp-noti (부모 5건, 답글 5건)
- 실패: 없음
```

## 자동 실행 (cron)

수동 호출(`/gcp-gmail`)이 기본. 정기 실행은 `schedule` skill로:

```
/schedule "매일 오전 9시와 오후 6시에 /gcp-gmail 실행"
```

자동 실행 시에도 신규 thread가 없으면 Slack 발송 생략.

## 에러 처리

- **Gmail MCP 미인증/장애**: `claude mcp list`로 상태 확인 안내. 자동 재시도 안 함.
- **Slack 채널 미존재**: `slack_search_channels`로 후보 조회 후 보고. config 수정 안내.
- **Slack rate limit / 5xx**: 30초 대기 후 1회 재시도. 그 이상은 사용자에게 보고하고 중단(처리한 만큼만 state 반영).
- **state.json 파싱 실패**: 사용자 확인 받고 백업 후 초기화. **자동 초기화 금지.**
- **부모는 성공 + 답글 실패**: 해당 thread를 state에 기록하지 않음. 사용자에게 명시 보고.

## 가드레일

- `config.yaml`을 사용자 승인 없이 수정하지 않습니다. (값 변경은 사용자가 직접 편집)
- `state.json`은 운영 데이터로 자유롭게 갱신.
- 메일 본문을 Slack에 통째로 옮기지 않습니다 — 핵심 요약·영향 프로젝트·액션만. 본문 전체가 필요하면 Gmail 링크로 이동.
- Gmail 라벨을 변경하지 않습니다 (이 skill은 라벨을 읽기만 함).
- 메일에 민감정보(키, 토큰, 비밀번호, 개인 식별번호)가 있어도 Slack에는 옮기지 않습니다.
- 부모 메시지를 묶지 않고 thread당 1개씩 보내므로, **수동 실행 시**에는 한 번에 부모가 10건을 넘을 것 같으면 먼저 사용자에게 "신규 N건인데 그대로 발송할까요?"라고 확인합니다. **자동(cron) 실행 시**에는 묻지 않고 그대로 발송하되, `max_threads_per_run` 상한을 지키고 결과 보고에 건수를 명시합니다.
- 메일 수신자(개인 / 조직 Essential Contact / 팀 별칭), 같은 공지의 다른 라벨 중복 수신 사실 — 이런 메타 정보는 부모/답글 어디에도 적지 않습니다. 사용자가 본문 내용으로 인사이트를 얻는 게 목적이며, 메타 정보는 그 목적을 흐립니다.

## 트리거 표현

- "/gcp-gmail"
- "gcp 메일 정리해줘"
- "메일 다이제스트 만들어줘"
- "라벨 X 메일 요약해서 슬랙으로"
