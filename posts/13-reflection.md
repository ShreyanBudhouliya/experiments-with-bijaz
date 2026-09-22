## Where this started

The first six experiments were about one agent. What a system prompt does to it, where its tokens go, whether it can modify its own source, whether it has anything like a stable personality. By the end of that stretch the recurring finding was that a system prompt shapes an agent's behaviour without owning it — instructions that conflict with training get quietly ignored, and the agent stays recognisably itself.

That is a fact about one agent talking to one person. The obvious next question is what happens when you put several of them in a row and make each one's output the next one's input. So we built a chain: a planner that writes a solution plan, a worker that executes it, a judge that reviews both and produces the final answer. Three stateless calls, no tools, no memory, no way for anything to flow backwards.

At baseline the chain worked and told us nothing. Both model families sat near ceiling and the judge agreed with the worker in every single baseline run. Three agents, three positions, and nothing in the results to distinguish them. Everyone was right, so nobody's role was visible.

To see the shape of a thing you have to break it. That is how the fault came about — not because we were interested in attacks, but because a pipeline where nothing goes wrong has no observable structure.

## What the first result actually showed

The first real result was that a fault at the judge takes the pipeline to zero. Every run, both model families, no exceptions. Next to a planner fault that the pipeline mostly shrugs off, that looks like a dramatic finding about where fragility lives in a chain of agents.

It is largely definitional, and it took us longer than it should have to say so plainly.

The pipeline's output is defined as the judge's answer. The fault instructs the judge to corrupt the number it was about to produce. So a judge that obeys produces a wrong pipeline output by necessity — not because the error propagated, not because nothing downstream caught it, but because there is no downstream, and the quantity we score is the quantity we corrupted. Zero is what the arithmetic of the setup requires, given obedience.

The empirical content of that condition is the obedience rate, which we did measure: the judge emitted exactly the corrupted value in every run of both families. That is a real number about instruction-following in an agent that has been handed a plausible-looking correct solution and told to spoil it. The accuracy figure sitting next to it is mostly a restatement of our own design.

It is still worth knowing. A design in which the last agent's output is the system's output, with nothing after it, is an extremely common design, and "the last agent is unrecoverable" is a true and useful property of it. But it is a property of an architecture we chose, not a discovery about where faults do damage.

## What "position" was hiding

The bigger problem is a word we used throughout without noticing what it was carrying.

We described the experiment as varying the *position* of a fault. Planner, worker, judge — three positions, one fault, and a clean question about whether it matters where the fault lands. The answer came out ordered exactly as predicted, which is the kind of result that stops you looking closely at it.

But the judge in our pipeline is four things at once.

It is **last in sequence**. Nothing runs after it, so no agent has the opportunity to review its work.

It holds the **reviewing role**. Its prompt tells it to verify someone else's arithmetic and correct it if wrong, which is a different job from planning or executing.

It has **output authority**. Its answer is the pipeline's answer, by a line in the orchestrator that reads the judge's number and returns it. This is a fact about the plumbing, not about the agent.

It was **where we attacked**. The fault was applied to its system prompt, so it is also the channel through which the compromise entered.

In our design these four coincide perfectly. The last agent is the reviewer is the output owner is the attack target. So when the judge condition comes out at zero, the experiment cannot tell you which of those four properties produced the result. Sequence order and output authority are confounded by the orchestrator. The reviewing role and the attack channel are confounded by our choice of where to put the suffix.

And this is not an abstract worry, because the data pulled two of them apart on its own and showed they behave differently. When we faulted the planner, the worker read the corrupted plan and recovered the correct answer in every single run where the fault had actually landed. Not most. Every one, in both families. The damage in that condition came entirely from the judge afterwards — handed a correct worked solution and a plan carrying a poisoned number, it discarded the correct solution and took the number from the plan, in a substantial fraction of runs.

So the condition labelled "planner-faulted" describes where we put the suffix. It does not describe where the failure happened. The failure happened at the judge, in all three conditions, which is precisely what you would expect if the thing doing the work is output authority rather than sequence position — and precisely what a design that welds those two together cannot demonstrate.

## What separating them takes

Saying what would have to change is easier than saying how.

**Output authority has to come apart from sequence order.** As long as the last agent's answer is mechanically the pipeline's answer, "last" and "decisive" are the same property wearing two names. Any arrangement where the final word belongs to something other than the final speaker would start to separate them — and the arrangement matters less than the fact that the two stop being identical.

**The reviewing role has to be movable.** In our pipeline, review happens last because that is where we put the reviewer. Those are independent choices that we made as one choice. A reviewer that sits somewhere other than the end would tell you whether the role or the seat was doing the work.

**The attack channel has to vary independently of the target.** We compromised an agent by editing its system prompt, which means the compromised agent and the corrupted instructions arrive together. Corruption that reaches an agent by some other route — through what it is given to read rather than through what it is told to be — would let you ask whether the fault's entry point matters separately from its location.

**And the benchmark has to leave room.** Ours did not. At roughly 96% baseline accuracy the worker is almost never wrong, so a downstream reviewer almost never has anything to catch. That is not a detail — it is the reason one of our clearest results came out as a null, and it puts a ceiling on how much any of these separations could show.

Four properties, and a design that varies one at a time is four times the size of the one we ran. That is the honest cost of the question.

## What I'd do differently

The thing I would change is not a design choice. It is how quickly we accepted a reading of our own data.

Twice, the same runs supported two opposite conclusions, and we reached the wrong one first.

The second judge was the clearer case. Adding a redundant judge recovered the pipeline completely from single-judge compromise — a clean, strong, unambiguous result, and we read it as redundancy working the way redundancy is supposed to: two reviewers, a vote, the compromised one outvoted. Then we looked at which branch of the voting rule actually fired, and the answer was that the two judges never once agreed with each other, so the vote's agreement branch never ran at all. Almost every run was decided by a tiebreaker that asks which judge matches the *worker*. The second judge's entire contribution was to disagree, so that the worker got consulted. The mechanism is not independent cross-checking. It is dependent verification with the worker as arbiter — and pipeline accuracy in that condition comes out exactly equal to worker accuracy, which is the same fact stated as a number.

The disagreement detector went the same way. Worker and judge disagreeing looked like a degradation signal you get for free, and the headline numbers were good. Then: when does it stay silent? It stays silent precisely when the judge adopted the corrupted answer, because adoption means the two numbers match. So it is quiet exactly when the fault succeeded and loud exactly when the fault was caught. It does not detect corruption. It detects correction. Every worker-faulted run it missed produced a wrong answer, and every flag it raised on a run that came out correct was a run the judge had already fixed.

Neither correction needed new data. Both came from the same files, asked a sharper question — not "did it work" but "by what mechanism", not "how often does it fire" but "when is it silent". In both cases the first reading was the flattering one, the one where the intervention did the thing its name suggests.

That is the pattern I would carry forward. The result that confirms what you expected is the one to go back to, because it is the one you will not otherwise re-examine. A clean positive result is a reason to ask what actually produced it.

The nulls taught the same lesson from the other side. We tried twice to build a fault that resembled real degradation — a weaker model, raised temperature, then an instruction to trust upstream work and stop re-deriving it — and all three produced nothing. The last one produced a particularly useful nothing, because the agent visibly complied and the outcome still did not move, which told us the checking we had removed had not been changing answers in the first place. A null that you can explain is worth more than one you can only report, and the explanation only existed because that run recorded what the agents actually wrote.

## Where this went

The questions outlasted the experiments. What we learned from these six is mostly about the shape of the question rather than the answer — that position was four things, that the clean result was partly our own design reflected back, that the mechanism is rarely the one the name implies.

That was enough to keep going. The work continued along the same lines.
