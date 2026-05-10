# CLAUDE.md — debate-conductor workspace

이 디렉터리는 **EP B — debate-conductor 변형** 데모 워크스페이스. Engine 은 `debate-conductor` plugin (별도 repo `pandas-studio/agent-team-plugins`). 이 워크스페이스는 topics 와 conductor 가 살 dir 만 제공.

| Pane | 역할 | 누가 |
|---|---|---|
| left  | Conductor / PM | Claude Code (이 세션) |
| middle | Generator 산물 | `gemini` (via plugin) |
| right | Critic 산물    | `codex` (via plugin) |

## 흐름

토론 요청을 받으면 plugin 의 세 skill 만 호출:

1. `/debate-conductor:bootstrap` — 한 번만. tmux 3-pane 분할 + role tail 시작.
2. `/debate-conductor:run <topic-number-or-name> [rounds]` — round 실행 + verdict 요약.
3. `/debate-conductor:continue [extra-rounds]` — 가장 최근 토론에 라운드를 N개 더 append (default 2). 같은 `debate-<TS>/` dir, 같은 토픽 (`topic.txt` 자동 인식). round 번호는 이어짐.

`debate.sh`/`ask-*.sh` 를 직접 부르지 말 것. Plugin 의 세 skill 이 entry point.

**`/run` vs `/continue` 판단**: 사용자가 "5라운드로 다시" 같이 말하면 새 토론 (`/run`). "더 돌려줘 / 이어서" 같이 말하면 같은 토론 연장 (`/continue`). 토픽이 다르면 무조건 `/run`.

## Verdict (round 2 마지막 줄)

| Verdict | 의미 |
|---|---|
| `STRENGTHEN` | 본질은 옳음 — round 3 에서 보강 |
| `RECONSIDER` | 핵심 가정 흔들림 — round 3 에서 가정 재논 |
| `OVERTURN` | 결론 뒤집어야 함 — round 3 에서 다른 각도 |

## Round 산물

`./.debate-conductor/log/<team>/latest-debate/round-<N>-{gen,crit}.md` (또는 `$DEBATE_LOG_DIR` 로 override). 인용·재현 가능한 디스크 아티팩트.

같은 dir 에 `topic.txt` (원 토픽, `/continue` 가 읽음) 와 hidden `.round-<N>-*.done` sidecar (완료 sentinel) 가 함께 산다. transcript 본문은 안 건드림 — 마지막 줄은 여전히 critic 의 `Verdict: STRENGTHEN|RECONSIDER|OVERTURN` (canonical 3-token).

## Reporting back

토론 끝나면 verdict 1줄 + round 별 한 줄 요약 + 후속 질문 2개. 긴 본문은 dump 하지 말 것 — 사용자는 middle/right pane 에서 이미 보고 있음.

## Don't

- 토론 요청을 main session 자체 분석으로 대체하지 말 것 — `/debate-conductor:run` 이 핵심.
- Topic 이 stance 없는 일반 질문이면 사용자에게 한 번 확인.
- `ask-generator.sh` / `ask-critic.sh` 직접 호출 금지 — plugin 의 `debate.sh` 만 entry point.
