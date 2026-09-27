# Jev Cheat Sheet

> State in. Typed probabilities out. The policy stays in your code.

Jev is TypeSafe's System One model. It does not write the reply, the summary, or the next line of a plan. You send state and typed questions. It returns a Choice, a Score, or a Noul, with probabilities. Choice and Score also return confidence.

Use this sheet when you already know you want a judgment inside software and you need the request shape, the fields, and the patterns that hold up past a single demo call.

Official docs: [docs.typesafe.ai](https://docs.typesafe.ai). This page follows those docs and the request shape below. Limits and prices move. The tables cite the docs as of Jev 1.13.

## Quick start

Python 3.10 or newer. The client reads `TYPESAFE_API_KEY` and calls `jev-latest` unless you pass a model.

```bash
pip install typesafe-sdk
```

```bash
# macOS or Linux
export TYPESAFE_API_KEY="your-key"
```

```powershell
# Windows PowerShell
$env:TYPESAFE_API_KEY = "your-key"
```

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

with TypeSafeClient() as client:
    response = client.system_one(
        state={
            "message": "The export button crashes Settings in Safari. Works in Chrome.",
            "plan": "team",
        },
        questions={
            "team": Choice(
                instructions="Which team should handle `message`?",
                criteria={
                    "frontend": "UI bugs and browser compatibility",
                    "backend": "API and server errors",
                    "devops": "Infrastructure and deployment",
                },
            ),
            "severity": Score(
                instructions="How severe is the reported issue in `message`?",
                criteria=[
                    "Cosmetic; no impact to functionality",
                    "Broken feature, but a workaround exists",
                    "Blocking issue; no workaround exists",
                ],
            ),
            "needs_hotfix": Noul(
                instructions="`message` describes an issue that should ship a hotfix before the next release.",
            ),
        },
    )

team = response.answers["team"]
print(team.choice, team.confidence, team.probabilities)
print(response.answers["severity"].score)
print(response.answers["needs_hotfix"].noul)
```

Keys: [console.typesafe.ai/keys](https://console.typesafe.ai/keys).

## Table of contents

- Level 1: The request
- Level 2: Choice, Score, Noul
- Level 3: Read the answer
- Level 4: State
- Level 5: One call, many questions
- Level 6: Policy in code
- Level 7: A second call
- Level 8: Models, the skill, and agents
- Level 9: Limits, price, errors
- Level 10: What to leave on an LLM
- Reference tables
- Troubleshooting

---

## Level 1: The request

Every call has three parts.

| Field | Role |
| --- | --- |
| `state` | The text, object, or array the questions judge |
| `questions` | A map of ids to Choice, Score, or Noul |
| `model` | `jev-latest` by default in the SDKs. The response returns the version that ran |

```bash
curl -s https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d @body.json
```

```powershell
$body = Get-Content .\body.json -Raw
Invoke-RestMethod `
  -Uri "https://api.typesafe.ai/v1/systemone" `
  -Method POST `
  -Headers @{ Authorization = "Bearer $env:TYPESAFE_API_KEY" } `
  -ContentType "application/json" `
  -Body $body
```

`body.json`:

```json
{
  "model": "jev-latest",
  "state": {
    "message": "My card was charged twice for the same order."
  },
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle `message`?",
      "criteria": {
        "billing": "Payment or subscription issues",
        "technical": "Bugs or integration problems",
        "sales": "Pricing or plan questions"
      }
    },
    "wants_refund": {
      "type": "noul",
      "instructions": "The customer is asking for money back."
    }
  }
}
```

The question id (`department`, `wants_refund`) is for your code. The model does not see it. Put the full meaning in `instructions` and `criteria`.

---

## Level 2: Choice, Score, Noul

Pick the primitive from the value your code will branch on.

| You need | Type | You get |
| --- | --- | --- |
| Exactly one of a set you defined | `choice` | `choice`, `probabilities`, `confidence` |
| A position on a rubric you wrote | `score` | `score`, `probabilities`, `confidence`, `legend` |
| Whether a statement holds | `noul` | `noul` from 0 to 1. No separate confidence field |

```python
from typesafe_sdk import Choice, Noul, Score

questions = {
    "department": Choice(
        instructions="Which team should handle `message`?",
        criteria={
            "billing": "Payment, subscription, or invoice problems",
            "technical": "Bugs, failed integrations, or API errors",
            "sales": "Pricing, plans, or a quote",
            "none": "Not a request any of those teams should own",
        },
    ),
    "frustration": Score(
        instructions="How frustrated does the customer appear in `message`?",
        criteria=[
            "Calm, just stating facts",
            "Frustrated but civil",
            "Very angry, strong language, or threatening to leave",
        ],
    ),
    "wants_human": Noul(
        instructions="The customer is asking to talk to a person.",
    ),
}
```

Choice criteria are a map. The key is the option your code will see. The value is the description that separates it from the others. A description may be `None` when the option name is already the whole meaning.

Score criteria are an ordered list. Index 0 is the low end. The score is a position on that list, not a class label. A result of `1.58` on a 0–2 rubric sits between level 1 and level 2.

Noul instructions are a statement the model can judge, written so that 1 means the statement holds.

```python
# statement, not a command to "analyze the ticket"
Noul(instructions="`message` reports something that is broken or failing.")
```

When a string criterion is too thin, use an object and keep the same fields on every option or level:

```python
Score(
    instructions="How severe is the reported issue?",
    criteria=[
        {"what": "Cosmetic; no impact to functionality", "examples": ["typo in a label"]},
        {"what": "Broken feature, but a workaround exists", "examples": ["export fails in Safari only"]},
        {"what": "Blocking; no workaround", "examples": ["checkout cannot be completed"]},
    ],
)
```

---

## Level 3: Read the answer

```python
answers = response.answers

department = answers["department"]
department.choice            # "billing"
department.confidence        # concentration of the distribution, 0 to 1
department.probabilities     # one probability per option you defined

frustration = answers["frustration"]
frustration.score            # position on the rubric
frustration.legend           # index to the level text
frustration.probabilities
frustration.confidence

answers["wants_human"].noul  # 0 to 1. There is no .confidence on a Noul
```

A Noul near `0.5` is a split between yes and no. It is not "medium urgency" and it is not a weak yes. Treat `>= 0.5` as a yes only when a coin flip is an acceptable mistake.

Confidence on a Choice or Score is how hard the probability mass sits on one option. Several acceptable options spread it out. Low confidence does not mean the call failed. It means your code should slow down.

```python
def band(department, wants_human: float) -> str:
    if wants_human >= 0.85:
        return "escalate"
    if wants_human >= 0.35 or department.confidence < 0.55:
        return "confirm"
    if department.choice == "none":
        return "confirm"
    return "route"
```

Those cutoffs are an example. Set them on your own tickets. Do not assert a vendor probability to two decimals in a test. Assert the branch your code takes when you pass a fake answer.

HTTP answers use the same ids:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "billing",
      "confidence": 0.91,
      "probabilities": {"billing": 0.94, "technical": 0.04, "sales": 0.01, "none": 0.01}
    },
    "wants_human": {"type": "noul", "noul": 0.08}
  },
  "usage": {"input_tokens": 420, "output_tokens": 71}
}
```

`usage.output_tokens` still shows up. Output tokens are not billed.

---

## Level 4: State

State is what you would put in front of a person before asking for a one-second judgment. A string is enough for one message. An object is the right call when the question must point at a field.

```python
def to_state(ticket: dict) -> dict:
    return {
        "message": ticket["message"],
        "plan": ticket.get("plan") or "unknown",
        "channel": ticket.get("channel") or "email",
        "prior_ticket_count": int(ticket.get("prior_ticket_count") or 0),
    }
```

If a field is not in that dict, the model cannot use it. If a secret is in that dict, the model receives it. Drop keys you would not paste into a prompt.

Reference a field with a backticked path inside the instructions: `` `message` ``, `` `ticket.plan` ``, `` `passages.stripe_reconnect` ``.

Supported state: a string, a JSON object, or an array of text values. Images, audio, and video are not inputs. Turn them into text first.

English is the strongest language. For anything else, test on your own text and read the confidence before you automate the branch.

---

## Level 5: One call, many questions

Questions in one request run together, against the same state, and cannot see each other's answers. Adding a question costs tokens. It barely changes latency. Ask the follow-ups you might need, then ignore the ones the first answer makes irrelevant.

```python
QUESTIONS = {
    "department": Choice(
        instructions="Which team should handle `message`?",
        criteria={
            "billing": "Payment, subscription, or invoice problems",
            "technical": "Bugs, failed integrations, or API errors",
            "sales": "Pricing, plans, or a quote",
            "none": "Not a request any of those teams should own",
        },
    ),
    "billing_issue": Choice(
        instructions="If `message` is a billing problem, which kind is it?",
        criteria={
            "double_charge": "Charged more than once",
            "failed_payment": "A payment or update did not go through",
            "invoice": "Wants an invoice or a receipt",
            "other": "None of the above",
        },
    ),
    "technical_area": Choice(
        instructions="If `message` is a product bug, where is it?",
        criteria={
            "integration": "A third-party connection such as Stripe",
            "api": "An API error or a 5xx",
            "ui": "A browser or screen problem",
            "other": "None of the above",
        },
    ),
}

def detail(answers) -> str | None:
    team = answers["department"].choice
    if team == "billing":
        return answers["billing_issue"].choice
    if team == "technical":
        return answers["technical_area"].choice
    return None
```

`billing_issue` is still answered when the team is technical. Your code is what throws it away. A second round trip that exists only to ask a question you could have included here is the expensive version of the same call.

Split a vague judgment the same way. "How bad is this ticket?" becomes severity, frustration, and how much an engineer can act on. Combine them in code.

---

## Level 6: Policy in code

Rubrics have different widths. A top score on a 3-level scale is `2`. A top score on a 4-level scale is `3`. Divide by the top index before you weight them.

```python
WEIGHTS = {"severity": 0.6, "frustration": 0.3, "report_quality": 0.1}

def normalized(answers, question_id: str, questions: dict) -> float:
    top = len(questions[question_id].criteria) - 1
    return answers[question_id].score / top

def priority(answers, questions: dict) -> float:
    # Severity is speculative. Ignore it when this is not a defect.
    terms = []
    if answers["is_defect"].noul >= 0.5:
        terms.append((WEIGHTS["severity"], normalized(answers, "severity", questions)))
    terms.append((WEIGHTS["frustration"], normalized(answers, "frustration", questions)))
    terms.append((WEIGHTS["report_quality"], normalized(answers, "report_quality", questions)))
    weight_sum = sum(weight for weight, _ in terms)
    return sum(weight * value for weight, value in terms) / weight_sum
```

Changing `WEIGHTS` reorders tickets without a new prompt. A weighted sum still routes a calm ticket that should never be automated. That case is a separate Noul your code checks before the score, not another term in the average.

```python
def action_for(answers) -> str:
    if answers["needs_a_person"].noul >= 0.8:
        return "escalate"
    return band(answers["department"], answers["wants_human"].noul)
```

Test the policy with constructed answers. A live call belongs in a batch you read, not in an equality assertion.

---

## Level 7: A second call

Make a second request when the first answer creates state you did not have, or when it chooses which options the next question is allowed to offer. Do not make one for a question that could have ridden along.

Passage text goes in state. The question only points at it.

```python
PASSAGES = {
    "stripe_reconnect": "Disconnect Stripe, wait one minute, connect it again from Settings.",
    "stripe_glossary": "Stripe is a payments company. This page does not describe a fix.",
}

def rank_passages(client, state: dict) -> list[tuple[str, float]]:
    questions = {
        passage_id: Noul(
            instructions=f"`passages.{passage_id}` helps an engineer act on `message`.",
        )
        for passage_id in PASSAGES
    }
    response = client.system_one(
        state={**state, "passages": PASSAGES},
        questions=questions,
    )
    ranked = [(key, response.answers[key].noul) for key in questions]
    ranked.sort(key=lambda item: item[1], reverse=True)
    return ranked

def judge(client, ticket: dict):
    state = to_state(ticket)
    first = client.system_one(state=state, questions=QUESTIONS)
    answers = first.answers
    team = answers["department"].choice
    passages = rank_passages(client, state) if team == "technical" else []
    return team, detail(answers), action_for(answers), passages
```

Sort by the Noul. A glossary page that shares the word Stripe should land under the page that tells an engineer what to do. A stop question, asked against the top of that list, is how you quit reading.

For a deep taxonomy, walk Choice level by level and keep more than the single winner when the probabilities are close. That is a second request for a real reason: the next option list did not exist until the parent was chosen.

---

## Level 8: Models, the skill, and agents

| Name | Use |
| --- | --- |
| `jev-latest` | Default. Alias. It moves when a stable release ships |
| `jev-preview` | Alias for the newest release, including a preview when one exists |
| `jev-1.13.0` | The version these docs currently point both aliases at. Pin this if your thresholds were tuned on it |

The response `model` field is the version that answered. Log it. `GET /v1/models` lists the aliases.

```python
from typesafe_sdk import TypeSafeClient

with TypeSafeClient() as client:
    for model in client.models.list().models:
        print(model.name, model.release_date, model.description)
```

```typescript
import { TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const models = await client.models.list();
for (const model of models) {
  console.log(model.name, model.release_date, model.description);
}
```

There is no per-account fine-tune. Domain behavior lives in `state`, `instructions`, and `criteria`. Customer requests are not used to train Jev.

Claude Code, and other agents, will default to one call per question and to a paragraph where a primitive belongs. The TypeSafe skill is the correction.

```bash
claude plugin marketplace add typesafe-ai/skills
claude plugin install typesafe@typesafe-ai
```

```bash
npx skills add typesafe-ai/skills --skill typesafe-ai
```

Use one install path. After it is installed, review the questions it writes. A generated question that bundles two judgments gets split before you merge it.

Jev decides. An LLM writes the sentence, and only after your gate says `route`. An escalate returns a handoff and no draft.

---

## Level 9: Limits, price, errors

From the models page for Jev 1.13. Rate limits are marked there as still moving under load.

| | |
| --- | --- |
| Endpoint | `POST https://api.typesafe.ai/v1/systemone` |
| Price | $0.042 per million input tokens, $42 per billion. Output tokens are free |
| Rate | 250,000 tokens per second, 1,200 requests per minute |
| Request budget | 64k tokens for state plus all questions |
| Longest single piece | 32k tokens for state plus the longest question |
| Input | Text. String, JSON object, or array of text |

```text
input_tokens / 1_000_000 * 0.042
```

420 input tokens is about $0.000018. Measure the `usage` object on your own call before you publish a ratio.

| Status | Meaning |
| --- | --- |
| 401 | Missing or bad key |
| 422 | Body failed validation. Read the field it names |
| 429 | Rate limit. SDKs back off. Raw HTTP should honor `retry-after` |
| 529 | Overloaded. Retry with backoff |

---

## Level 10: What to leave on an LLM

Send Jev the judgments your code can name in advance:

- route, rank, score, flag, stop, verify a candidate you already extracted
- a yes/no over a passage, a field, or a trace
- a closed list, including a `none` option when nothing may fit

Leave these on a model that can write:

- the customer reply, the commit message, the explanation
- a plan, a proof, or anything that needs a chain of reasons
- a string that was not in your criteria and was not extracted by code first

A type-safe answer is still allowed to be wrong. The schema cannot invent an option you did not list. It can pick the wrong one. Verification is a second judgment, or a person, not a feeling that the JSON parsed.

---

## Reference tables

### Which primitive

| Situation | Use |
| --- | --- |
| One team, one product, one language, including "none of these" | Choice |
| Severity, frustration, relevance, quality, on a ladder you describe | Score |
| Urgent, refund, defect, "this passage helps", "stop searching" | Noul |
| Several labels can all be true | One Noul each, not one Choice |
| A number you will weight | Score, then normalize, then weight in code |
| A number you will threshold | Noul, and you pick the threshold |

### Answer fields

| Primitive | Fields your code should read |
| --- | --- |
| Choice | `choice`, `probabilities`, `confidence` |
| Score | `score`, `probabilities`, `confidence`, `legend` |
| Noul | `noul` |

### Patterns

| Pattern | Do this |
| --- | --- |
| Fan-out | Every independent question in the first request. Ignore unused answers |
| Composite score | One Score per dimension. Normalize. Weight in code |
| Hard override | A Noul checked before the weighted score. A serious case is not averaged away |
| Conditional detail | Ask the follow-up anyway. Read it only on the matching branch |
| Select, then verify | Code or an LLM proposes a value from evidence. A Choice or Noul checks it |
| Rerank | One Noul per candidate. Sort by the probability. Then a stop question |
| Hierarchy | Next Choice uses the children of the option that won |
| Pin the model | Log `response` model. Pin the version id when thresholds are tuned |

---

## Troubleshooting

| What you see | What to check |
| --- | --- |
| 401 | `Authorization: Bearer` and `TYPESAFE_API_KEY` in the environment the process actually inherited |
| 422 | `type` is `choice`, `score`, or `noul`. Choice `criteria` is an object. Score `criteria` is an array |
| `confidence` missing on a Noul | Expected. Read `noul` |
| Follow-up question looks random | It did not see the team answer. Either fold it into the instructions as "if this is billing..." or ignore it in code after you know the team |
| Two scores will not compare | You weighted them before dividing by `len(criteria) - 1` |
| A calm legal threat gets routed | The weighted sum cannot express "any serious hit escalates". Add a Noul and check it first |
| Tests flake | You asserted a live probability. Assert the branch on a fake answer |
| The agent made one HTTP call per question | One `questions` map. Install the TypeSafe skill and review the diff |
| Non-English text feels off | Expected to be weaker. Test your corpus and gate on confidence |
| Huge state, weaker answers | The 64k and 32k budgets, and accuracy that shifts as state grows. Send the fields the questions name |

---

## Contributing

Open a pull request with the docs page or the call that backs the change. Prices, rate limits, and model ids go stale. If you update one, name the docs page and the date in the commit.

## License

MIT. See [LICENSE](LICENSE).

## Resources

- [Documentation index](https://docs.typesafe.ai/llms.txt)
- [Quick start](https://docs.typesafe.ai/introduction/quickstart)
- [Choice](https://docs.typesafe.ai/primitives/choice), [Score](https://docs.typesafe.ai/primitives/score), [Noul](https://docs.typesafe.ai/primitives/noul)
- [Confidence](https://docs.typesafe.ai/confidence)
- [Python SDK](https://docs.typesafe.ai/sdk/python)
- [JavaScript SDK](https://docs.typesafe.ai/sdk/javascript)
- [API](https://docs.typesafe.ai/api)
- [Models](https://docs.typesafe.ai/models)
- [Agent skill](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)
