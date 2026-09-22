## What we're doing and why

The first six experiments were all one agent. One agent with different system
prompts, one agent writing its own tools, one agent burning tokens, one agent
answering the same questions five times over. That was fun, and it ran out of
questions.

The next one was obvious enough: what is it like to experiment on a chain of
them? Several agents in a row, each one's output becoming the next one's input.
Nothing in the first six tells you what that does — whether errors cancel out
along the chain or travel down it, whether the agents behave differently when
they are reading each other instead of a person.

So we built one. This experiment builds the chain and checks that it works,
before we start breaking it.

## What we're building

The earlier experiments all ran bijaz on coding tasks, where "did it work" is a
judgement call and half the results table ends up saying "kinda". For a pipeline
we wanted the opposite: a task with one right answer, checkable by an equality
test, so that any change in the output is a change we can point at.

So the pipeline solves grade-school math word problems. A planner reads the
problem and writes a solution plan without computing anything. A worker executes
the plan and produces a number. A judge reviews both and produces the final
number.

Conditions:
* Each agent is a single stateless API call
* No tools, no conversation history, no memory between runs
* Everything an agent knows arrives in its prompt; everything it contributes leaves in its output text
* Each agent sees only what the previous ones chose to write down

This is the simplest structure that still has the property we care about:
information passes between agents, and it passes as text.

The three roles are a deliberately conventional shape. Plan, execute, review is
roughly what agent frameworks converge on, and the judge-at-the-end pattern is
the standard way people add reliability to an LLM system. Cursor describes
running planning, execution and verification stages this way in
[Scaling long-running autonomous coding](https://cursor.com/blog/scaling-agents).

## Setup

* Three-agent pipeline: planner → worker → judge
* Models: `claude-sonnet-4-6` and `gpt-5.4`, run as two separate families
* Within a family, all three agents use the same model
* One API call per agent, three per pipeline run
* No tools, no conversation history, no context carryover between agents or between runs
* Temperature left at each provider's default; no generation seed set
* 30 tasks × 3 runs per task × 2 families = 180 baseline runs

Because temperature is at provider default and no seed is set, re-running this experiment will not reproduce these numbers run-for-run. The three runs per task exist to give us a spread, not to give us repeatability.

## The three roles

### Planner

**Receives:** the problem text, and nothing else.

**Produces:** a step-by-step plan — the quantities involved, the operations, the order to do them in. It is told explicitly not to compute the final answer.

<details>
<summary>Planner system prompt</summary>

```
You are a careful math problem-solver. Given a word problem, write out a plan for solving it step by step. 
Identify the quantities, operations, and the order of computation. Do NOT compute the final answer — just produce the solution plan.
```

</details>

### Worker

**Receives:** the problem text and the planner's full output, concatenated.

**Produces:** the worked computation, ending with a number in the GSM8K canonical format `#### <number>`.

<details>
<summary>Worker system prompt</summary>

```
You are a math problem-solver. You will receive a word problem and a solution plan. 
Execute the plan carefully, showing each computation. End your response with "#### <final_number>" on its own line.
```

</details>

### Judge

**Receives:** the problem text, the planner's full output, and the worker's full output.

**Produces:** a verification. If the worked solution is right it restates the answer; if wrong it identifies the error and produces a corrected answer. Same `#### <number>` format.

<details>
<summary>Judge system prompt</summary>

```
You are a math reviewer. You will receive a word problem, a solution plan, and a worked solution. 
Verify the arithmetic and reasoning. If correct, restate the answer. If incorrect, identify the error and produce the correct answer. End your response with "#### <final_number>" on its own line.
```

</details>

## Information flow

Who sees what:

| Agent | Sees the problem | Sees the plan | Sees the worked solution |
|---|---|---|---|
| Planner | Yes | — | — |
| Worker | Yes | Yes | — |
| Judge | Yes | Yes | Yes |

Every agent sees the original problem. Nothing is hidden from downstream agents, and nothing flows backwards — the planner never learns what the worker did with its plan.

Two details matter more than they look.

**The pipeline's output is the judge's answer.** Not a vote, not a consensus, not the worker's number with the judge as a check. We take whatever number the judge wrote and that is the answer the pipeline gives. The judge is the last agent and it is also the only agent whose output leaves the system.

**There is one fallback.** If no integer can be extracted from the judge's text, the orchestrator uses the worker's answer instead. This is the only path by which a non-judge answer can become the pipeline's output.

We also extract a number from the planner's output and record it, even though the planner is told not to compute one. This is an extraction artifact, not a real prediction — `extract_answer` falls back to the last integer in the text, so a planner that writes out an intermediate quantity will "have" an answer. It is useful only as a trace of what the planner was thinking about.

## How the orchestrator runs

<details>
<summary>run_pipeline, from orchestrators.py</summary>

```python
# Step 1: Planner
planner_result = run_single_agent(
    task=task,
    system_prompt=planner_config["system_prompt"],
    agent_id="planner",
    ground_truth=None,
    model=planner_config["model"],
)

# Step 2: Worker (receives task + planner output)
worker_task = f"Task: {task}\n\nPlan from planner:\n{planner_result.output}"
worker_result = run_single_agent(
    task=worker_task,
    system_prompt=worker_config["system_prompt"],
    agent_id="worker",
    ground_truth=None,
    model=worker_config["model"],
)

# Step 3: Judge (receives task + plan + worked solution)
judge_task = (
    f"Task: {task}\n\n"
    f"Plan:\n{planner_result.output}\n\n"
    f"Worked solution:\n{worker_result.output}"
)
judge_result = run_single_agent(
    task=judge_task,
    system_prompt=judge_config["system_prompt"],
    agent_id="judge",
    ground_truth=None,
    model=judge_config["model"],
)

step_results = [planner_result, worker_result, judge_result]
step_answers = [r.extracted_answer for r in step_results]

# Final answer: judge's answer, falling back to worker's if judge returned None
final_answer = judge_result.extracted_answer
if final_answer is None:
    final_answer = worker_result.extracted_answer

correct = (ground_truth is not None) and (final_answer == ground_truth)
```

</details>

Note that `ground_truth=None` is passed to every agent. No agent is ever told whether it got the problem right. Correctness is computed once, at the end, outside the pipeline.

## Task used

GSM8K, the grade-school math word problem benchmark. `tasks.py` loads the test split, shuffles it at seed 42, and keeps the first 50 items as a fixed sample. `experiment.py` then sets `TASK_INDICES = list(range(30))` and uses the first 30 items of that shuffle.

So these 30 tasks are not a 30-item random sample of GSM8K. They are a prefix of a 50-item sample. The 50 were drawn at a fixed seed; the 30 are whichever of those happened to land first in the shuffle. Anything we claim about task-level variation is a claim about these 30 specific problems.

Answers are extracted by `extract_answer` in `tasks.py`, which tries `#### <number>` first, then a set of phrase patterns ("answer is", "answer:", trailing `=`, bolded number), then falls back to the last integer in the text.

Why GSM8K rather than a coding task like the earlier experiments used: we need a single unambiguous integer per run so that "did the pipeline get it right" is a mechanical check rather than a judgement call. Every earlier experiment in this series had "task completed: yes / kinda / partial" rows in its results table. We wanted to stop doing that.

## Metrics

Recorded per run:

* Final answer (integer or null)
* Correct (final answer equals ground truth)
* Step answers — the extracted integer from each of the three agents
* Total cost across all three calls
* Total duration across all three calls

Cost is recomputed in `runner.py` from a price table rather than read from the API response, so these are list-price estimates, not billed amounts.

Ground truth is not stored in the result rows. For the numbers below we recovered it per task by taking the final answer of any run marked correct.

This works for 29 of the 30 tasks. Task 24 has no correct run anywhere in the file — all 24 of its runs are wrong — so its ground truth cannot be recovered this way. Task 24 is also the task both families fail on in the results table below, which means the one task we can say least about is the one doing the most work in that table.

## Result: does it work

| Metric | sonnet | gpt |
|---|---|---|
| Baseline runs | 90 | 90 |
| Correct | 87 | 86 |
| Accuracy | 96.7% | 95.6% |
| Tasks correct on all 3 runs | 29 / 30 | 28 / 30 |
| Tasks correct on no run | 1 / 30 | 1 / 30 |
| Tasks with mixed results | 0 / 30 | 1 / 30 |
| Mean cost per run | $0.0107 | $0.0081 |
| Median cost per run | $0.0106 | $0.0075 |
| Cost range per run | $0.0058 – $0.0230 | $0.0035 – $0.0199 |
| Total baseline cost | $0.96 | $0.73 |
| Mean duration per run | 10.5s | 8.4s |
| Duration range | 6.8s – 19.4s | 5.0s – 15.8s |

Both families fail on the same single task, all three runs, both times. For sonnet that accounts for every failure it has. For gpt it accounts for three of four.

## Observations

### the judge and the worker never disagreed
* In all 180 baseline runs the judge's extracted answer equals the worker's extracted answer. Not "usually" — 180 out of 180.
* The worker is already right about 96% of the time, so there is very little for the judge to correct.
* A judge that agrees with a correct worker is behaving exactly as specified. This is not evidence that the judge is broken.
* It is evidence that at baseline the judge contributes no independent signal we can measure. Its answer is fully determined by the worker's.

### the fallback never fired
* Across all 720 runs in the file — baseline and every faulted condition — the judge produced an extractable integer.
* The worker-fallback path in the orchestrator has never executed.
* It stays in the code, but nothing we report depends on it.

### the planner's recorded answer is mostly noise
* The planner's extracted number matched ground truth in 4 of 90 sonnet runs and 0 of 90 gpt runs.
* That is what should happen. The prompt tells it not to compute an answer.
* The 4 sonnet cases are `extract_answer` grabbing a trailing intermediate value, not the planner solving the problem.
* The planner column of `step_answers` is a trace, not a prediction.

### the two families write different plans
* On the first task, sonnet's planner trace lands on the problem's stated total in all three runs.
* gpt's planner trace lands on a much smaller intermediate in two of three runs, and on the total in the third.
* Both families' workers produced the correct answer every time.

### gpt is cheaper and faster, at slightly lower accuracy
* About 24% cheaper per run and about 20% faster.
* 1.1 percentage points lower accuracy — one run, on one task.
* That gap is well inside the noise of 90 runs with no seed. We are not claiming a difference between the families.

### the ceiling is high enough to be a problem
* 96.7% and 95.6% leave three and four failing runs respectively to reason about.
* Any effect we want to measure by making the pipeline better has almost no room to show up.
* This is workable for what comes next, because what comes next makes it worse.

## What this sets up

The pipeline works, both families are near ceiling, and a run costs about a cent. That gives us a system that is reliable enough that any drop in accuracy is attributable to something we did, and cheap enough to run several hundred times.

It also gives us the thing we actually wanted, which is a chain with a shape. There are three positions. Each sees a different amount of the problem. Only one of them speaks for the pipeline. Experiment 6 showed that a system prompt shapes an agent's behaviour without fully controlling it — the next question is what happens to a chain when one link's system prompt is shaped against us, and whether it matters which link.

What this experiment does not establish:

* That the judge does anything. 180 out of 180 agreements with the worker is consistent with a working reviewer and also consistent with a judge that does nothing. We can't tell them apart from baseline data, because a correct worker gives a judge nothing to do.
* That three agents beat one. We ran no single-agent control here, so we have no idea whether the planner and judge are earning their two-thirds of the cost.
* That these accuracy numbers generalise. 30 problems, all from the front of one seeded shuffle of one benchmark, all short-form arithmetic with an integer answer.
* That the numbers are reproducible. No seed, provider-default temperature. A re-run would land somewhere near these figures, not on them.
* Anything about the pipeline under stress. Every run here is a run where nothing was wrong.
