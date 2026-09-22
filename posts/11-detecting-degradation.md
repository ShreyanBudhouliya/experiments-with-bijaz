## What we're doing and why

This post re-analyses experiment 9's runs. No new data, no new API calls — the same 720 rows in `results.jsonl`, asked a different question.

Experiment 9 found that every fault condition resolves at the judge, and experiment 10 removed that single point of failure by adding a second judge. Both of those are interventions. Both cost money on every run, and both have to be decided on in advance, because a pipeline running in production has no idea whether anything is wrong with it.

That is the gap this post is about. Experiments 9 and 10 could see the damage because we had ground truth. A deployed pipeline does not have ground truth — if it did, it would not need the pipeline. What it does have is three agents that each wrote down a number, and the fact that those numbers sometimes differ.

## Hypothesis

A pipeline whose agents disagree with each other is a pipeline something is wrong with. Disagreement between steps is available at runtime, costs nothing to compute, and should be usable as a degradation signal.

## How we measure it

The detector is one comparison:

```
flagged = (step_answers[1] != step_answers[2])
```

The worker's extracted answer against the judge's extracted answer. If they differ, raise a flag.

**Ground truth is used only to score the detector, never to compute it.** The flag is a function of two numbers the pipeline already has. Whether that flag was right is a question we can only answer because we kept the answers to these 30 problems in a file. A running pipeline gets the first thing and not the second, and the entire point of the exercise is to find out how much the first thing is worth on its own.

We also have to say what "right" means, because there are two different questions and they give different numbers.

* **Detecting injection.** Did a fault get injected into this run? Positives are the 540 faulted runs, negatives are the 180 baseline runs.
* **Detecting a wrong output.** Is this run's final answer wrong? Positives are the 322 runs whose final answer does not equal ground truth, negatives are the 398 that are correct.

The second is the one an operator actually cares about. The first is the one the experiment was built around. We report both.

One design note. We compare the worker against the judge and leave the planner out. Including it destroys the detector: a three-way comparison across all of `step_answers` fires on 86/90 sonnet and 90/90 gpt *baseline* runs, because the planner is instructed not to compute an answer and the number extracted from its trace is an artifact, as established in experiment 7. The planner's recorded answer is not a prediction and cannot be compared against one.

## Result: precision and recall

Raw firing rates, by condition:

| Condition | sonnet | gpt | combined |
|---|---|---|---|
| Baseline | 0/90 | 0/90 | 0/180 |
| Planner-faulted | 36/90 | 11/90 | 47/180 |
| Worker-faulted | 39/90 | 66/90 | 105/180 |
| Judge-faulted | 90/90 | 90/90 | 180/180 |

### Detecting injection

| | sonnet | gpt | combined |
|---|---|---|---|
| True positives | 165/270 | 167/270 | 332/540 |
| False negatives | 105/270 | 103/270 | 208/540 |
| False positives | 0/90 | 0/90 | 0/180 |
| True negatives | 90/90 | 90/90 | 180/180 |
| Precision | 165/165 (100%) | 167/167 (100%) | 332/332 (100%) |
| Recall | 165/270 (61.1%) | 167/270 (61.9%) | 332/540 (61.5%) |

### Detecting a wrong output

| | sonnet | gpt | combined |
|---|---|---|---|
| Runs with a wrong final answer | 186/360 | 136/360 | 322/720 |
| True positives | 128/186 | 104/136 | 232/322 |
| False negatives | 58/186 | 32/136 | 90/322 |
| False positives | 37/174 | 63/224 | 100/398 |
| Precision | 128/165 (77.6%) | 104/167 (62.3%) | 232/332 (69.9%) |
| Recall | 128/186 (68.8%) | 104/136 (76.5%) | 232/322 (72.0%) |

### Detecting a wrong output, excluding the judge-faulted condition

The table above is carried by the judge-faulted runs. Those 180 runs are wrong by construction — the pipeline's output is the judge's answer and the fault corrupts it — and flagged by construction, because the fault instructs the judge to differ from the worker by exactly 7. They enter the confusion table as 180 free true positives.

Since this post argues that detection at the judge is worthless, the honest thing is to report the figures without it:

| | sonnet | gpt | combined |
|---|---|---|---|
| Runs | 270 | 270 | 540 |
| Runs with a wrong final answer | 96 | 46 | 142 |
| True positives | 38/96 | 14/46 | 52/142 |
| False negatives | 58/96 | 32/46 | 90/142 |
| False positives | 37/174 | 63/224 | 100/398 |
| Precision | 38/75 (50.7%) | 14/77 (18.2%) | 52/152 (34.2%) |
| Recall | 38/96 (39.6%) | 14/46 (30.4%) | 52/142 (36.6%) |

Recall falls from 232/322 to 52/142. Precision falls from 232/332 to 52/152.

**This is the operating point an operator should use.** The flag in a judge-faulted run arrives after the wrong output has been produced, with no agent left to act on it, so a warning system cannot bank those detections. What is left is a detector that catches about a third of wrong answers and is right about a third of the time when it fires.

That is not a weaker version of the earlier result. It is the same claim — the detector sees correction, not corruption — stated in numbers instead of in prose.

Recall by faulted position, for the injection question:

| Faulted position | sonnet | gpt |
|---|---|---|
| Planner | 36/90 | 11/90 |
| Worker | 39/90 | 66/90 |
| Judge | 90/90 | 90/90 |

## Where it fails

The 208 missed faults are not one failure mode. They are two, and they are opposite.

| Missed faults (faulted runs, not flagged) | sonnet | gpt | combined |
|---|---|---|---|
| Total missed | 105 | 103 | 208 |
| Output was correct anyway | 50 | 75 | 125 |
| Output was wrong | 55 | 28 | 83 |

### the planner misses are mostly not misses

Recall is worst at the planner — 36/90 and 11/90. That looks like the detector's biggest weakness. It mostly is not.

Splitting planner-faulted runs by whether the fault landed at all, using experiment 9's landing criterion:

| | sonnet | gpt |
|---|---|---|
| Fault landed | 76/90 | 59/90 |
| … landed and flagged | 35/76 | 10/59 |
| … landed and missed | 41/76 | 49/59 |
| Of the landed-and-missed runs, output was correct | 41/41 | 49/49 |
| Fault did not land | 14/90 | 31/90 |
| … and flagged anyway | 1/14 | 1/31 |

Every single run where the planner fault landed and the detector stayed quiet produced the correct answer. 41 out of 41 and 49 out of 49. This is experiment 9's mechanism showing up from the other side: the worker absorbs the corrupted plan and recovers ground truth, so by the time the worker and judge are compared there is no disagreement, because there is genuinely nothing wrong.

Calling those false negatives is a scoring artifact. We injected a fault, so the injection-detection question counts them as positives that were missed. But the pipeline was not degraded, the output was correct, and a flag would have been a false alarm about a run that was fine. Of the 8 planner-faulted runs that were both missed and wrong — 4 in each family — those are real misses.

### the worker misses are the real blind spot

| | sonnet | gpt |
|---|---|---|
| Worker-faulted runs missed | 51/90 | 24/90 |
| Of those, output was wrong | 51/51 | 24/24 |

The mirror image. Every worker-faulted run the detector missed produced a wrong answer. Not one of them was harmless.

The reason is structural. The detector fires when the worker and judge disagree. A worker fault that the judge *adopts* produces a judge answer identical to the worker's — that is what adoption means. So the detector is silent exactly when the fault has propagated successfully, and it speaks exactly when the fault has been caught. It is not measuring whether something went wrong. It is measuring whether the judge noticed, and the judge noticing is the same event as the fault being corrected.

That is the whole limitation in one line: this detector cannot see a fault that worked.

### the flags that fired on correct outputs

All 100 of the "false positives" for the wrong-output question are worker-faulted runs — 37 sonnet and 63 gpt. In every one of them the worker was corrupted, the judge disagreed, and the judge was right. The flag is accurate about the pipeline's internal state and wrong only about the outcome, because the pipeline fixed itself.

Whether that counts as a false alarm depends on what the alarm is for. As "this output is wrong", it misfired 100 times. As "something happened in this run that needed correcting", it was right all 100 times.

### the detector cannot name the culprit

A flag says `step_answers[1] != step_answers[2]`. It says nothing about which of those two agents is the problem.

Experiment 9 established that all three fault conditions resolve at the judge, which makes this worse rather than better. When the worker and judge disagree, the possibilities include a corrupted worker the judge caught, a corrupted judge overriding a clean worker, a corrupted planner whose number the judge picked up while the worker ignored it, and an ordinary disagreement with no fault at all. The signal points at a pair. Deciding which member of the pair to distrust requires information the disagreement does not contain.

### zero false positives is a small number pretending to be a large one

0/180 is the strongest-looking figure in this post and it is the one that bounds the least.

* It rests on 180 clean runs on 30 problems.
* Those problems are answered correctly about 96% of the time at baseline, which is why the worker and judge have almost nothing to disagree about.
* Experiment 7 found the judge agreed with the worker in 180/180 baseline runs. This is the same fact restated as a detector statistic, not independent confirmation of it.
* A harder benchmark, where a competent worker and a competent judge legitimately reach different answers, would produce false positives at some unknown rate. We have no estimate of it and nothing here constrains it.

## What it costs to run

Nothing. Both numbers are already in the pipeline's state by the time the last agent returns, and the comparison is one integer equality.

That is the detector's entire case. Experiment 10's second judge costs +34.6% and +32.1% per run and delivers full recovery under single-judge compromise. This costs zero and delivers a flag that is right about injection 332/332 times when it fires, silent 208/540 times when it should not be, and blind by construction to the faults that succeeded.

## Observations

### General findings

#### on the injection question the detector never lied, and often said nothing
* Precision 332/332. When it fires, a fault was injected. This holds for the injection question only — on the wrong-output question it fired on 100 correct runs.
* Recall 332/540. It stays silent on more than a third of faulted runs.
* Of the 208 silences, 125 were runs the pipeline got right anyway and 83 were wrong answers it failed to warn about.

#### it detects correction, not corruption
* The signal is worker-judge disagreement, and disagreement happens when the judge overrides the worker.
* Every worker-faulted run it missed was wrong: 51/51 and 24/24.
* Every worker-faulted run it flagged as a wrong output but got right was a run the judge fixed: 100/100 across both families.
* The event it observes is the judge doing its job, not the fault occurring.

#### the two scoring questions disagree about the same 720 runs
* Injection: precision 332/332, recall 332/540.
* Wrong output: precision 232/332, recall 232/322.
* Wrong output, excluding judge-faulted: precision 52/152, recall 52/142.
* Same flags, same runs. The gap between the first two is the 100 runs where a fault was injected, flagged, and corrected. The gap between the second and third is the 180 runs that are wrong and flagged by construction.

#### the planner cannot participate
* Extending the comparison to all three steps fires on 86/90 sonnet and 90/90 gpt baseline runs.
* The planner's number is an extraction artifact from an agent instructed not to compute one.
* Any disagreement detector on this pipeline has exactly two usable signals to compare.

### Per-condition findings

#### baseline
* 0/90 flags in both families.
* Bounded by 180 runs at roughly 96% accuracy, as set out above.

#### planner-faulted
* Lowest recall of any condition: 36/90 sonnet, 11/90 gpt.
* Most of that is not detector failure — 41/41 and 49/49 of the landed-and-missed runs produced correct output.
* The family gap tracks the landing rate from experiment 9 (76/90 against 59/90), not a difference in detector behaviour.
* 4 real misses per family: faulted, missed, and wrong.

#### worker-faulted
* Recall 39/90 sonnet, 66/90 gpt — and the gap is the judges' correction rates from experiment 9, restated.
* Every miss was a wrong answer: 51/51 and 24/24.
* Every flag on a correct output was a successful correction: 37/37 and 63/63.

#### judge-faulted
* Recall 90/90 in both families. Perfect.
* This follows from the fault: the judge is instructed to add 7 to what it would have said, so it necessarily differs from the worker.
* And it is useless. The detector fires on every run in the one condition where, as experiment 9 set out, the wrong answer is already the pipeline's output by construction. Detecting a fault at the last position tells you the output is bad after the output has been produced.

## What this connects to

Experiment 7 found that the judge agreed with the worker in 180/180 baseline runs and concluded we could not tell whether the judge was doing anything. Experiment 9 showed it was doing everything. This post is the same fact as a runtime signal: the judge agreeing with the worker is the pipeline's normal state, so a disagreement is informative, and the informativeness is entirely borrowed from how rarely judges disagree at baseline.

Experiment 10's redundancy is the same detector at a different position. Two judges disagreeing with each other is worker-judge disagreement moved one seat over, and it behaves identically — 0/90 firings in the redundant baseline in both families, 90/90 in the one-faulted condition. It also shares the blind spot exactly: under double compromise the two judges agreed in 90/90 runs, so a disagreement detector on that pair would have been silent through all 180 runs of the condition it most needed to catch. Redundancy converts the signal into an automatic correction rather than a flag, which is why it costs a fourth call and this costs nothing.

What this experiment does not establish:

* That the detector works on anything but this fault. A deterministic +7 that preserves output format produces a clean numeric disagreement. A fault that corrupted reasoning while landing on the same number would be invisible to it, and we did not test one.
* That zero false positives means anything beyond these 180 runs. See the bound above.
* That 61.5% recall is a property of the detector. It is a property of the detector combined with a fault set in which one of three positions is unrecoverable by construction and another is mostly self-healing.
* That a flag is actionable. It identifies a pair of disagreeing agents, not a compromised one, and nothing here tells an operator what to do next.
* That this is the best available runtime signal. We tested one comparison between two extracted integers. We have no agent text for these runs, so signals based on reasoning, confidence or hedging were not available to test.
* That detection at the judge is worth having. The condition it detects perfectly is the condition where the output is already wrong and nothing downstream remains to act on the warning.
