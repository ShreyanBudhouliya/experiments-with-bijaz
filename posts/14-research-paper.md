## What happened

The six experiments before this one were meant to be a detour. Build a chain of
agents, break it in one place at a time, see what the chain does. It turned into
about a year of work, and the work became a paper.

It is called "When the Judge Intervenes: Instruction Injection and Evidence
Poisoning in Sequential LLM Pipelines", and it is here:

[shreyan.xyz/research/when-the-judge-intervenes](http://shreyan.xyz/research/when-the-judge-intervenes)

## What it does that these posts didn't

Experiment 13 ended on the problem with this series: the word "position" was
carrying four things at once. The judge was last in sequence, held the reviewing
role, owned the pipeline's output, and was where we put the fault. All four
coincided, so none of the results could say which one mattered.

The paper separates them. It attacks through channels other than the agent's own
instructions, on a task where the answer depends on evidence the agent is given
rather than arithmetic it can redo, and it compares what a pipeline does against
what a single agent does on the same items. Once those things vary independently,
position by itself stops being the operative variable.

It also keeps the part of this series that survived scrutiny and drops the part
that did not. The judge-faulted result that opened experiment 9 is not a finding
in the paper. It is one paragraph of motivation, described as what it is.

## On these posts

Everything in experiments 7 through 13 stands as written. The numbers are the
numbers, and the two corrections in experiment 13 are corrections I would rather
publish than quietly fix.

The paper is where the questions went. These posts are where they started.
