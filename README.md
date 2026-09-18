# delegate

A [Claude Code](https://claude.com/claude-code) skill that grades a pile of tasks by risk and hands
each group to the model that actually fits it — Haiku, Sonnet, or Opus.

*[한국어 설명은 아래에](#한국어)*

## The problem

When several tasks pile up, running all of them through one model goes wrong in both directions.
Put everything on the expensive model and you burn it on typo fixes. Put everything on the cheap
model and something subtle gets quietly broken. The judgment call gets made from vibes, differently
every time.

## What it does

It does not just recommend a model. It uses the Agent tool's `model` parameter to actually hand the
work over. Orchestration — grading, grouping, verification — stays in the main session; the file
edits happen inside the chosen model's subagent.

| Grade | Criteria | Model |
|---|---|---|
| Clerical | Mechanical, almost no judgment, easy to reverse | `haiku` |
| Standard | Needs an understanding of surrounding logic and existing patterns | `sonnet` |
| High-risk | Hard to reverse, or wide blast radius if wrong | `opus` |

When in doubt it grades **up**, never down — a bad edit costs more than a better model.

## The part that matters: it refuses to delegate

Most of this skill is about *not* delegating. A subagent starts with **zero knowledge of the
conversation**, and that cost is invisible until it bites — an agent with no context does not stop
when it gets stuck, it invents something plausible and edits the wrong thing.

So before anything is handed off, the skill checks:

- **Is there enough work?** 1–2 tasks, or tasks that depend on each other, are faster done directly.
- **Is each task self-contained?** "Fix that thing we talked about" cannot go to an agent that
  wasn't there. Either make it concrete first, or keep it.
- **Does grouping leave more than one group?** Tasks touching the same file must go in one call, so
  ten items that all land in a single stylesheet are one group — nothing runs in parallel, and the
  only benefit left is context savings. If the session already read that file, even that is gone.
  **The threshold is groups after grouping, not item count.**
- **Is it one indivisible deep task?** Then it recommends switching the session model instead, and
  says so in one line rather than forcing a split.

It also never trusts a subagent's "done" — it verifies per grade (look at it / run the tests / read
the diff), retries one grade up on failure, and stops and reports rather than looping.

And it does not commit. Ever, unless asked.

## Install

```bash
git clone https://github.com/RealLight04/claude-delegate.git ~/.claude/skills/delegate
```

That path makes it a user-level skill, available in every project. For a single project instead,
clone into `.claude/skills/delegate/` inside that project.

Claude Code picks it up without a restart.

## Usage

Invoke it explicitly:

```
/delegate
```

Optionally pass the list directly:

```
/delegate fix the three a11y findings in the audit above
```

With no arguments it uses the task list built up in the conversation. It also triggers on its own
when a list of 3+ independent to-dos has just been assembled — and deliberately stays quiet for a
single one-off instruction.

## Requirements

- Claude Code, with the Agent tool available (it needs the `model` parameter to delegate)
- Nothing else. It is a single `SKILL.md`.

## License

MIT — see [LICENSE](LICENSE).

---

## 한국어

할 일이 여러 개 쌓였을 때, 작업마다 위험도를 판단해서 맞는 모델(Haiku/Sonnet/Opus)에 실제로
위임하는 Claude Code 스킬입니다.

### 왜 필요한가

작업이 쌓였을 때 전부 같은 모델로 돌리면 양쪽으로 잘못됩니다. 비싼 모델에 다 맡기면 오타
수정에 그걸 쓰게 되고, 싼 모델에 다 맡기면 미묘한 걸 조용히 망칩니다. 그 판단이 매번 감으로,
매번 다르게 내려집니다.

이 스킬은 추천만 하지 않습니다. Agent 도구의 `model` 파라미터로 실제 그 모델에 작업을 넘깁니다.
등급 판단·묶기·확인은 메인 세션이 하고, 파일 수정은 고른 모델의 서브에이전트가 합니다.

| 등급 | 기준 | 모델 |
|---|---|---|
| 사무적 | 기계적이고 판단이 거의 없으며 되돌리기 쉬움 | `haiku` |
| 일반 | 주변 로직·기존 패턴을 이해해야 함 | `sonnet` |
| 고위험 | 되돌리기 어렵거나 파급이 큼 | `opus` |

애매하면 **위로** 올립니다. 잘못 고치는 비용이 모델 비용보다 크기 때문입니다.

### 핵심은 "위임하지 않는 판단"입니다

이 스킬의 대부분은 사실 *위임하지 않는* 조건에 관한 것입니다. 서브에이전트는 **대화 맥락을
하나도 모른 채** 시작하고, 그 비용은 사고가 나기 전까지 안 보입니다 — 맥락 없는 에이전트는
막히면 멈추는 게 아니라 그럴듯하게 지어내서 엉뚱한 걸 고칩니다.

그래서 넘기기 전에 확인합니다.

- **양이 충분한가** — 1~2개거나 서로 의존적인 작업은 직접 하는 게 빠릅니다.
- **각 작업이 자기완결적인가** — "아까 얘기한 그거"는 그 자리에 없던 에이전트에 못 넘깁니다.
  먼저 구체화하거나, 아니면 직접 합니다.
- **묶고 나서 그룹이 2개 이상인가** — 같은 파일을 건드리는 작업은 한 호출에 묶어야 하므로,
  10개 항목이 전부 CSS 파일 하나로 모이면 그룹은 1개입니다. 병렬이 안 되니 남는 이득은 컨텍스트
  절약뿐이고, 세션이 그 파일을 이미 읽었다면 그것도 없습니다. **기준은 항목 수가 아니라 묶은 뒤의
  그룹 수입니다.**
- **쪼갤 수 없는 깊은 작업인가** — 그렇다면 억지로 쪼개지 않고 세션 모델을 바꾸라고 한 줄로
  추천합니다.

서브에이전트의 "완료했습니다"도 그대로 믿지 않습니다. 등급별로 확인하고(눈으로 / 테스트 실행 /
diff 정독), 실패하면 한 등급 올려 재시도하고, 그래도 안 되면 무한 재시도 대신 멈추고 보고합니다.

그리고 커밋은 하지 않습니다. 요청받기 전까지는.

### 설치

```bash
git clone https://github.com/RealLight04/claude-delegate.git ~/.claude/skills/delegate
```

이 경로에 두면 사용자 레벨 스킬이 되어 모든 프로젝트에서 쓸 수 있습니다. 특정 프로젝트에서만
쓰려면 그 프로젝트의 `.claude/skills/delegate/`에 클론하세요. 재시작 없이 바로 인식됩니다.

### 사용

`/delegate`로 직접 부르거나, 뒤에 목록을 붙여도 됩니다. 인자 없이 부르면 대화에서 쌓인 작업
목록을 씁니다. 독립적인 할 일이 3개 이상 정리된 직후에는 알아서 발동하고, 단발 지시 하나에는
일부러 발동하지 않습니다.
