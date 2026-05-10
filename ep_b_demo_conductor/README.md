# EP B 실습 — debate variant (Claude conductor)

> EP B 영상 라이브 데모 재현. **Claude (PM)** 가 `gemini` (Generator) 와 `codex` (Critic) 를 N-round adversarial 토론에 동원. 화면은 가로 3-pane — 좌측 Claude, 중앙 generator 산물, 우측 critic 산물.

🎬 **영상**: (TODO — EP B 업로드 후 채움)

**사전 준비**: Claude Code · Codex · Gemini 모두 설치/인증 → [SETUP.md](../SETUP.md)

---

## Step 1. Plugin 마켓플레이스 등록

먼저 아무 디렉터리에서 `claude` 한 번 띄우고:

```
/plugin marketplace add pandas-studio/agent-team-plugins
/plugin install debate-conductor@pandas-studio
```

설치 끝나면 Claude 는 종료해도 됨. plugin 은 영구 등록됨.

---

## Step 2. 워크스페이스 준비

```bash
git clone https://github.com/pandas-studio/agent-harness-tutorial
cp -r agent-harness-tutorial/ep_b_demo_conductor /tmp/ep-b-conductor-demo
cd /tmp/ep-b-conductor-demo
git init -q && git add . && git commit -q -m "init"
```

`CLAUDE.md` + `topics/` 만 있는 가벼운 워크스페이스.

---

## Step 3. 토픽 둘러보기

```bash
ls topics/
# 01-sqla-migration.txt  02-monorepo-vs-polyrepo.txt  03-server-actions-vs-rsc.txt

cat topics/01-sqla-migration.txt
```

각 파일은 generator 가 받을 **stance-driven 토픽** — 일반 질문이 아니라 한쪽 입장을 강요하는 형태라 토론이 깊어진다.

---

## Step 4. tmux + Claude 띄우기

```bash
tmux new-session -s debate
# tmux 안에서
claude
```

Claude 가 뜨면, 이 워크스페이스의 `CLAUDE.md` 와 plugin 의 두 skill 이 자동 로드됨.

---

## Step 5. Conductor bootstrap

Claude 안에서:

```
/debate-conductor:bootstrap
```

현재 pane 이 좌·중·우 3 칸으로 갈라지면서 중앙·우측에 `tail-role.sh gen|crit` 가 자동 시작. 좌측엔 그대로 Claude 세션이 남음.

> 💡 **tmux prefix 안내**: 기본 prefix 는 `Ctrl-b`. pane 이동 `Ctrl-b` → `←/→`, 분할 직접 `Ctrl-b` → `%`.

✅ 화면:

```
┌─────────────────┬─────────────────┬─────────────────┐
│ Claude          │ ── GENERATOR ── │ ── CRITIC ──    │
│ (conductor)     │ waiting for     │ waiting for     │
│ > _             │ debate to start │ debate to start │
└─────────────────┴─────────────────┴─────────────────┘
```

---

## Step 6. 토론 실행

```
/debate-conductor:run 1
```

또는 자연어로 "토픽 2번으로 5라운드 토론 돌려줘" — Claude 가 `$ARGUMENTS` 해석해서 같은 skill 호출.

진행 중에는:
- 좌측 Claude pane: "debate.sh 실행 중" 도구 호출 표시
- 중앙: gemini 가 한 줄씩 generator 본문을 흘려보냄
- 우측: codex 가 critic 본문 + 마지막에 verdict 한 줄

3-10 분 후 토론 끝나면 좌측 Claude 가:
- Verdict 한 줄 (STRENGTHEN / RECONSIDER / OVERTURN)
- Round 별 한 줄 요약 (R1 gen 핵심 → R2 crit 공격 → R3 gen 응답)
- 후속 질문 2 개

---

## Step 7. 토론 이어가기 — `/continue`

같은 토픽으로 라운드를 더 쌓고 싶으면:

```
/debate-conductor:continue 2
```

또는 자연어 "2 라운드 더 추가해줘". `/run` 과 다른 점:

- 새 `debate-<TS>/` 를 만들지 않고 **같은 dir 안에** `round-4-crit.md`, `round-5-gen.md` 추가 (round 번호 이어감).
- 토픽은 `topic.txt` (`/run` 시 자동 기록) 에서 자동 인식.
- 중앙·우측 pane 은 새 round 가 추가되는 즉시 흘려보냄 (`── more rounds appended — re-tailing ──` 짧은 separator 후).
- 새 라운드의 generator 는 **즉전 한 사이클** (직전 gen + 직전 crit) 만 컨텍스트로 받음 — 전체 히스토리 누적이 아니라 prompt 가 일정.

---

## Step 8. 후속 대화

```
> round 3 결정타가 뭐야?
> 5라운드로 다시 돌려줘            ← 새 토론 (/run)
> 2 라운드 더 추가해줘              ← 같은 토론 이어감 (/continue)
> rotate ON 으로 토픽 2 한 번 더
```

자연어로 conductor 와 대화. 재실행은 자동으로 같은 skill 재호출.

---

## 산물 위치

```
/tmp/ep-b-conductor-demo/.debate-conductor/log/<team>/
├── latest-debate -> debate-<TS>
├── debate-<TS>/
│   ├── topic.txt                     # /continue 가 자동 읽음
│   ├── round-1-gen.md
│   ├── round-2-crit.md
│   ├── round-3-gen.md                # /run 끝
│   ├── round-4-crit.md               # /continue 가 추가
│   └── round-5-gen.md                # /continue 가 추가
└── gen-<TS>.log, crit-<TS>.log
```

각 round file 첫 줄에 `<!-- debate-round: N role model -->` machine-readable marker 가 있고, `ls -la` 시 hidden `.round-N-*.done` sidecar 가 보일 수 있다 — `/continue` 가 어느 round 까지 완료됐는지 안전하게 추적하기 위한 device. transcript 본문엔 영향 없음 (마지막 줄은 여전히 critic 의 `Verdict: ...`).

다른 워크스페이스로 옮기거나 `DEBATE_LOG_DIR` 로 override 가능.

---

## 자주 막히는 곳

- `/debate-conductor:bootstrap` 이 "not in tmux" 에러 — 먼저 `tmux new-session -s debate` 후 `claude`.
- 중앙·우측 pane 이 "waiting for debate to start..." 에서 안 움직임 — 정상. `/run` 호출 직후 파일이 생기면 흘러내림.
- `ask-generator: gemini: command not found` — `gemini` CLI 인증 안 됨. [SETUP.md](../SETUP.md).

## 비교 — split variant

같은 토론을 셸 직접 호출로 보고 싶으면 sibling 시드 [`ep_b_demo_split`](../ep_b_demo_split) — `debate.sh` 를 사용자가 좌측 셸에서 직접 친다. Conductor 변형은 그 위에 Claude PM layer 를 얹은 것.
