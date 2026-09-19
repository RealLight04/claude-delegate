# delegate

[![License: MIT](https://img.shields.io/github/license/RealLight04/claude-delegate)](LICENSE)

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

할 일이 여러 개 쌓였을 때, 작업마다 위험도를 판단해서 맞는 모델(Haiku/Sonnet/Opus)에 실제로
위임하는 Claude Code 스킬임.

### 왜 필요한가

작업이 쌓였을 때 모든 일을 단일 모델로 처리하면 두 가지 문제가 생김.

- **고성능 모델 전담**: 단순 오타나 포맷팅 같은 작업에 비싼 모델을 쓰게 돼서 비용이 낭비됨.
- **경량 모델 전담**: 미묘하고 복잡한 로직을 경량 모델에 맡기면 오류를 제대로 못 잡고
  코드를 조용히 망가뜨림.

결국 "어떤 모델에 맡길 것인가"에 대한 판단이 매번 직관에 의존해서 일관성 없이 내려짐.

이 스킬은 단순한 모델 추천에 그치지 않음. 등급 판단·그룹화·검증은 메인 세션이 하고, 실제
작업은 Agent 도구의 `model` 파라미터로 지정한 모델의 서브에이전트에 **직접 위임**함.

| 등급 | 판단 기준 | 할당 모델 |
|---|---|---|
| 사무적 | 기계적이며 판단 요소가 적고, 되돌리기(rollback) 쉬운 작업 | `haiku` |
| 일반 | 주변 로직 및 기존 코드 패턴에 대한 이해가 필요한 작업 | `sonnet` |
| 고위험 | 되돌리기 어렵거나 프로젝트 전체에 파급력이 큰 작업 | `opus` |

**판단 원칙**: 등급이 애매하면 상위 모델로 올려서 배정함. 모델 사용 비용보다 잘못 수정된
코드를 복구하는 비용이 훨씬 크기 때문.

### 핵심은 "위임하지 않을 조건"을 구별하는 것

이 스킬의 핵심은 역설적으로 "어떤 작업을 위임하지 않을 것인가"에 있음.

서브에이전트는 이전 대화 맥락을 전혀 모른 채 작업에 투입됨. 맥락이 부족한 에이전트는 작업이
막혔을 때 멈추는 게 아니라, 그럴듯한 환각(hallucination)을 만들어내며 엉뚱한 코드를 수정하곤
함. 이로 인한 부작용은 사고가 터지기 전까지 눈에 안 띔.

그래서 작업을 서브에이전트에 넘기기 전에 다음 조건부터 확인함.

- **작업의 규모** — 1~2개 수준의 소규모 작업이거나 상호 의존성이 높은 작업은 메인 세션이
  직접 처리하는 게 빠름.
- **자기완결성** — "아까 말한 그것"처럼 이전 맥락이 필요한 요청은 위임 불가. 요청을 완전히
  구체화한 뒤 넘기거나, 직접 처리해야 함.
- **그룹화 후 최종 그룹 수** — 동일한 파일을 수정하는 작업들은 하나의 호출로 묶어야 함.
  예를 들어 작업 10개가 결과적으로 CSS 파일 하나로 모이면 최종 그룹은 1개. 이 경우 병렬
  처리가 불가능해서 컨텍스트 절약 외에는 이득이 없고, 메인 세션이 이미 그 파일을 읽었다면
  그 이득조차 없음. **핵심 기준은 개별 항목 수가 아니라 묶인 뒤의 최종 그룹 수(2개 이상).**
- **분할 불가능한 고난도 작업** — 억지로 쪼개지 않고, 메인 세션의 모델 자체를 전환하도록
  한 줄로 추천함.

### 검증 및 예외 처리

서브에이전트의 "완료했습니다"라는 응답을 그대로 신뢰하지 않음.

1. **단계별 검증** — 등급에 따라 검증 수준을 차등 적용함(눈으로 확인 / 테스트 실행 /
   diff 정독).
2. **단계적 승격** — 작업이 실패하면 한 등급 높은 모델로 재시도함.
3. **무한 재시도 방지** — 그래도 계속 실패하면 무한 루프에 빠지지 않고 즉시 중단한 뒤
   사용자에게 보고함.
4. **커밋 제한** — 자동 커밋은 절대 안 함.

### 설치

```bash
git clone https://github.com/RealLight04/claude-delegate.git ~/.claude/skills/delegate
```

이 경로에 두면 사용자 레벨 스킬이 돼서 모든 프로젝트에서 쓸 수 있음. 특정 프로젝트에서만
쓰려면 그 프로젝트의 `.claude/skills/delegate/`에 클론. 재시작 없이 바로 인식됨.

### 사용

`/delegate`로 직접 부르거나, 뒤에 목록을 붙여도 됨. 인자 없이 부르면 대화에서 쌓인 작업
목록을 씀. 독립적인 할 일이 3개 이상 정리된 직후에는 알아서 발동하고, 단발 지시 하나에는
일부러 발동 안 함.

### 예시 출력

작은 앱을 코드 리뷰했더니 문제 4개가 나왔다고 하면, `/delegate`는 이렇게 등급을 매기고
보고함.

1. **`utils.py`의 미사용 import 제거** — 사무적 → `haiku`
   기계적이고 판단 요소 없음. **완료.**
2. **페이지네이션 off-by-one 수정 (`api.py`)** — 일반 → `sonnet`
   주변 반복문 로직 이해 필요. **완료.**
3. **널 체크 누락 추가 (`user.py`)** — 일반 → `sonnet`
   호출부의 가정을 이해해야 함. **완료.**
4. **결제 웹훅 재시도 로직 재작성 (`billing.py`)** — 고위험 → `opus`
   결제 경로라 되돌리기 어려움. **완료** — 승인 전 diff 직접 검토.

파일이 4개로 갈리니 그룹도 4개 — 네 작업 모두 병렬로 처리되고, 세션이 마침 쓰던 모델이
아니라 각 위험도에 실제로 맞는 모델로 처리됨.
