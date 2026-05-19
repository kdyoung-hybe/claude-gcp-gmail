# claude-gcp-gmail

GCP 알림 메일(`gcp-no-reply`, `gcp-no-reply-actionrequired` 라벨)을 정기적으로 정리해 Slack으로 발송하는 Claude Code skill의 **클라우드 실행본**.

## 구성

- **SKILL.md** — skill 동작 명세 (Claude가 따라가는 절차)
- **config.yaml** — 라벨, Slack 채널, 요약 옵션
- **state.json** — 처리한 thread ID와 마지막 실행 시각 (실행마다 routine이 갱신·commit·push)

## 실행 방식

이 repo는 [claude.ai/code/routines](https://claude.ai/code/routines)에 등록된 schedule routine이 매시 정각(UTC)에 clone하여 사용합니다.

- Routine이 clone → SKILL.md 절차 수행 → state.json 갱신 → 자동 commit & push
- 사용자의 로컬 머신(맥)과 무관하게 Anthropic Cloud에서 실행

수동 실행은 로컬에서 `/gcp-gmail` 호출로 (로컬 `~/.claude/skills/gcp-gmail/` 사본 사용).

## 보안 메모

- 토큰·비밀번호·키 없음. Gmail / Slack 접근은 claude.ai의 MCP connector OAuth로 이루어짐
- `state.json`에는 Gmail thread ID와 타임스탬프가 누적됨 (활동 메타데이터)
