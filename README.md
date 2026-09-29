# delegate

[![License: MIT](https://img.shields.io/github/license/RealLight04/claude-delegate)](LICENSE)

A [Claude Code](https://claude.com/claude-code) skill that grades a pile of tasks by risk and hands
each group to the model that actually fits it — Haiku, Sonnet, or Opus.

*[한국어 설명은 아래에](#한국어)*

> New to Claude Code? A **skill** is a reusable prompt Claude Code loads on request (here, via
> `/delegate`). A **subagent** is a fresh Claude instance this skill spawns to do one task — it
> starts with none of the current conversation's context, which is exactly the risk this skill
> is built to manage.

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
the diff), retries once at the same grade with a tighter scope, then once one grade up, and stops
and reports rather than looping.

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

## Example output

A code review turns up four issues in a small app. `/delegate` grades each one and reports back
like this:

1. **Remove unused import in `utils.py`** — Clerical → `haiku`
   Mechanical, no judgment call. **Done.**
2. **Fix off-by-one in pagination (`api.py`)** — Standard → `sonnet`
   Needs the surrounding loop logic. **Done.**
3. **Add missing null check (`user.py`)** — Standard → `sonnet`
   Needs to understand caller assumptions. **Done.**
4. **Rewrite payment webhook retry logic (`billing.py`)** — High-risk → `opus`
   Payment path, hard to reverse if wrong. **Done** — diff reviewed before accepting.

Four different files means four groups, so all four ran in parallel — each on the model that
actually fit the risk, not whichever model happened to be driving the session.

## Requirements

- Claude Code, with the Agent tool available (it needs the `model` parameter to delegate)
- Nothing else. It is a single `SKILL.md`.

## License

MIT — see [LICENSE](LICENSE).

---

## 한국어

할 일이 여러 개 쌓였을 때 작업마다 위험도를 따져서 알맞은 모델(Haiku, Sonnet, Opus)에게 실제로 넘기는
Claude Code 스킬입니다.

> Claude Code를 처음 써본다면 두 단어만 알고 가면 됩니다. 스킬은 필요할 때 불러 쓰는 프롬프트로,
> 여기서는 `/delegate`로 부릅니다. 서브에이전트는 이 스킬이 작업 하나를 맡기려고 새로 띄우는 Claude입니다.
> 지금 대화에서 무슨 얘기가 오갔는지 전혀 모르는 채로 시작합니다. 이 스킬은 바로 그 위험을 다루려고
> 만들었습니다.

### 왜 필요한가

작업이 쌓였는데 전부 한 모델로 돌리면 어느 쪽으로든 탈이 납니다. 비싼 모델에 다 맡기면 오타 하나 고치는
데도 비싼 값을 치릅니다. 싼 모델에 다 맡기면 까다로운 로직을 티 안 나게 망가뜨립니다. 그러다 보니 어느
모델에 맡길지를 그때그때 감으로 정하게 되고 기준도 매번 달라집니다.

이 스킬은 모델을 추천하는 데서 끝나지 않습니다. 등급을 매기고 작업을 묶고 결과를 검증하는 건 메인 세션이
맡고 실제 파일 수정은 Agent 도구의 `model` 파라미터로 고른 모델의 서브에이전트가 합니다.

| 등급 | 기준 | 모델 |
|---|---|---|
| 사무적 | 기계적이고 판단할 게 거의 없으며 되돌리기 쉬운 작업 | `haiku` |
| 일반 | 주변 로직과 기존 코드 패턴을 이해해야 하는 작업 | `sonnet` |
| 고위험 | 되돌리기 어렵거나 잘못되면 파급이 큰 작업 | `opus` |

등급이 애매하면 항상 위로 올립니다. 잘못 고친 코드를 되살리는 비용이 더 좋은 모델을 쓰는 비용보다 훨씬
큽니다.

### 넘기지 않을 작업을 가려내는 게 절반입니다

이 스킬은 넘기는 것만큼이나 넘기지 않는 판단에 공을 들입니다. 서브에이전트는 대화 맥락을 하나도 모른 채
투입되는데 그 대가는 사고가 나기 전까지 잘 보이지 않습니다. 맥락 없는 에이전트는 막혀도 멈추지 않고
그럴듯한 답을 지어내서 엉뚱한 곳을 고칩니다.

그래서 넘기기 전에 네 가지를 먼저 봅니다.

- 일이 충분히 많은가. 작업이 한두 개뿐이거나 서로 얽혀 있으면 메인 세션이 직접 하는 편이 빠릅니다.
- 작업 하나하나를 따로 떼어 놓아도 이해할 수 있는가. "아까 말한 그거 고쳐줘"는 그 자리에 없던 에이전트에게
  넘길 수 없습니다. 먼저 구체적으로 풀어서 쓰거나 직접 처리합니다.
- 묶고 나서도 그룹이 두 개 이상인가. 같은 파일을 건드리는 작업은 한 번의 호출로 묶어야 합니다. 작업 열
  개가 전부 스타일시트 하나로 모이면 그룹은 하나뿐이라 병렬로 돌릴 게 없고 남는 이득은 컨텍스트 절약
  정도인데 메인 세션이 그 파일을 이미 읽었다면 그마저도 없습니다. 그래서 기준은 항목 수가 아니라 묶은 뒤의
  그룹 수입니다.
- 쪼갤 수 없는 어려운 작업 하나인가. 그렇다면 억지로 나누지 않고 세션 모델을 바꾸라고 한 줄로 권합니다.

### 검증과 실패 처리

서브에이전트가 끝났다고 보고해도 그 말만 믿지 않습니다. 확인하는 강도는 등급마다 달라서 사무적 작업은
결과를 직접 보고 일반 작업은 테스트까지 돌리며 고위험 작업은 diff를 직접 읽어 본 다음에 받아들입니다.

확인에서 문제가 나오면 먼저 같은 모델에게 범위를 더 좁혀서 한 번 더 맡깁니다. 그래도 안 되면 한 등급 위
모델로 한 번 더 시도합니다. 거기서도 안 되면 계속 붙잡고 있지 않고 멈춘 뒤 무엇을 해 봤는지 보고합니다. 커밋은 요청받기 전에는 절대 하지 않습니다.

### review-fix와 무엇이 다른가

같은 사람이 만든 [review-fix](https://github.com/RealLight04/review-fix-skill)도 등급을 매겨 모델을 고르고
결과를 검증합니다. 둘을 가르는 건 무엇을 입력으로 받느냐입니다.

delegate는 서로 독립적인 할 일 목록이면 종류를 가리지 않습니다. 기능 추가든 리팩터링이든 리뷰 결과든
넘길 수 있습니다. 그래서 넘기기 전에 이 작업이 맥락 없이도 이해되는지와 묶은 뒤 그룹이 몇 개 남는지를
먼저 따집니다. review-fix는 코드 리뷰가 이미 찾아낸 finding만 받습니다. 파일과 줄과 문제가 정해져 있으니 위임할지
말지를 따질 필요가 적고 대신 검증 안 된 finding 거르기, `CLAUDE.md`에 그 규칙이 정말 있는지 확인하기,
리뷰 뒤에 파일이 바뀐 finding 건너뛰기처럼 리뷰 수정에 필요한 장치가 더 촘촘합니다.

정리하면 delegate는 범용 위임기이고 review-fix는 그중 리뷰 finding 처리만 떼어 더 깊게 판 버전입니다.

상황별로 보면 이렇습니다.

- PR에 `/code-review`를 돌렸더니 finding이 여섯 개 나왔습니다. 안 쓰는 import 둘, 빠진 널 체크 하나,
  결제 재시도 버그 하나, 확인이 덜 된 PLAUSIBLE 둘입니다. 이럴 땐 review-fix를 씁니다. 확인된 네 개만
  등급별 모델로 고치고 PLAUSIBLE 두 개는 목록으로만 돌려줍니다.
- 배포 전에 README 오타, 설정 화면 다크 모드 버그, 로그 정리 스크립트 추가, 결제 모듈 리팩터링이
  쌓였습니다. 리뷰에서 나온 게 아니고 성격도 제각각이니 delegate를 씁니다. 파일이 모두 달라 네 그룹으로
  나뉘고 병렬로 처리됩니다.
- 할 일이 한두 개뿐이면 delegate를 쓸 필요가 없습니다. finding이 사소한 것 하나라면 review-fix도
  마찬가지입니다. 그냥 직접 시키는 편이 빠릅니다.

### 설치

```bash
git clone https://github.com/RealLight04/claude-delegate.git ~/.claude/skills/delegate
```

이 경로에 두면 사용자 레벨 스킬이 되어 모든 프로젝트에서 쓸 수 있습니다. 한 프로젝트에서만 쓰려면 그
프로젝트의 `.claude/skills/delegate/`에 클론하세요. 재시작하지 않아도 바로 인식합니다.

### 사용

`/delegate`로 직접 부르거나 뒤에 작업 목록을 붙여서 부릅니다. 인자 없이 부르면 대화에서 쌓인 작업 목록을
씁니다. 독립적인 할 일이 세 개 이상 정리된 직후에는 알아서 나서지만 지시가 하나뿐일 때는 일부러 가만히
있습니다.

### 예시 출력

작은 앱을 코드 리뷰했더니 문제가 네 개 나왔다고 해 봅시다. `/delegate`는 하나씩 등급을 매기고 이렇게
보고합니다.

1. `utils.py`의 안 쓰는 import 제거 → 사무적, `haiku`
   판단할 게 없는 기계적 작업입니다. 완료.
2. `api.py` 페이지네이션 off-by-one 수정 → 일반, `sonnet`
   주변 반복문 로직을 알아야 합니다. 완료.
3. `user.py`에 빠진 널 체크 추가 → 일반, `sonnet`
   호출하는 쪽이 무엇을 가정하는지 알아야 합니다. 완료.
4. `billing.py` 결제 웹훅 재시도 로직 재작성 → 고위험, `opus`
   결제 경로라 잘못되면 되돌리기 어렵습니다. diff를 직접 검토한 뒤 완료.

파일이 네 개로 갈리니 그룹도 네 개입니다. 네 작업은 모두 병렬로 처리됐고 세션이 어떤 모델로 돌고
있었든 각 작업은 위험도에 맞는 모델이 맡았습니다.
