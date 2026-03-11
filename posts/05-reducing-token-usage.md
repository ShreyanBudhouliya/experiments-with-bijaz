## What we're doing and why
* Experiment 4 showed us where bijaz's tokens go — 40% was a single python script resent 5 times, 81% of context was dead weight by the final call. 
* This experiment tests three techniques to reduce that waste, and compares them against the baseline.

## What techniques are used and why

From reading anthropic's context engineering blog and the jetbrains research paper on efficient context management, we identified two main approaches the industry uses to manage growing context:

1. **Observation Masking**: 
    * Replace old tool outputs with short placeholders. 
    * Used by SWE-agent (princeton/stanford).
    * The Jetbrains paper ("the complexity trap") found this halved costs while matching the performance of more complex approaches.

2. **LLM Summarisation**: 
    * Make an extra API call to summarise the conversation, then replace old messages with the summary. 
    * Used by Claude Code (Anthropic's Auto-Compact), Cursor, and Warp.
    * This is the industry default for production coding agents.

3. **Truncation**: 
    * We added a third, simpler technique as a floor.
    * It's the most naive approach: just cut long outputs short. no extra API calls, no intelligence, just a character limit. 
    * We wanted to see whether the simplest possible intervention could compete with the more sophisticated ones.

## Techniques (Code Snippets)
We experimented with 3 techniques:

### 1.Truncate
#### 1.1 What it does: 
  * Caps every tool result at 500 characters. 
  * Big bash outputs or large file reads get cut off before they enter the conversation history. 
  * Claude sees less per turn, so the context grows slower — but it may also miss information in the truncated part.
    
#### 1.2 Code snippet:
<details>
    <summary>Before each tool result is added to the message
         list:</summary>
    
    if TOKEN_STRATEGY == "truncate":                                                                       
          original_len = len(result)                                                                         
          if original_len > 500:
              result = result[:500] + f"\n[truncated — full output was {original_len} chars]"                
</details>

### 2.Mask
#### 2.1 What it does: 
  * Keeps the last 6 messages (3 turns) intact and replaces all older messages with a short 80-char placeholder. 
  * The original messages list is never modified — only the copy sent to the API is masked. So future calls still have the full history available to mask fresh each time. 
  * This reduces tokens sent per call without losing recent context.
    
#### 2.2 Code snippet:
<details>
    <summary>In call_api, before sending to the API:</summary>
    
    if TOKEN_STRATEGY == "mask":
          cutoff = max(0, len(messages) - 6)                                                                 
          api_messages = []                                                                                  
          for i, msg in enumerate(messages):                                                                 
              if i < cutoff:                                                                                 
                  preview = (content if isinstance(content, str) else str(content))[:80]                     
                  api_messages.append({"role": msg["role"], "content": f"[masked turn — ...]"})              
              else:                                                                                          
                  api_messages.append(msg)                
</details>

### 3.Summarise
#### 3.1 What it does: 
  * Estimates the conversation's token size. If it's over 3000 tokens and long enough to compress, it fires a second API call asking Claude to produce a concise summary of everything except the last 4 messages. 
  * It then collapses the history in-place to just that summary plus the most recent messages. 
  * The extra tokens from the summarization call are tracked in the session totals and printed as [summarize: ...].
    
#### 3.2 Code snippet:
<details>
    <summary>In call_api, before sending to the API:</summary>
    
    elif TOKEN_STRATEGY == "summarize":
        total_tokens = sum(len(str(msg["content"])) // 4 for msg in messages)
        if total_tokens > 3000 and len(messages) > 4:
            to_summarize = messages[:-4]
            summary_prompt = (
                "Summarize this agent conversation concisely. Preserve: file paths created, "
                "key decisions, errors encountered and how they were resolved, and current task state. "
                "Discard: full file contents, verbose command outputs, planning text.\n\n"
                + "\n".join(
                    f"{m['role']}: {str(m['content'])[:500]}" for m in to_summarize
                )
            )
            try:
                summary_response = client.messages.create(
                    model="claude-sonnet-4-5-20250929",
                    max_tokens=1000,
                    messages=[{"role": "user", "content": summary_prompt}],
                )
                summary_text = summary_response.content[0].text
                sum_in = summary_response.usage.input_tokens
                sum_out = summary_response.usage.output_tokens
                sum_cost = (sum_in / 1_000_000 * 3) + (sum_out / 1_000_000 * 15)
                total_input_tokens += sum_in
                total_output_tokens += sum_out
                total_cost += sum_cost
                print(
                    f"{DIM}[summarize: +{sum_in} in / +{sum_out} out | ${sum_cost:.3f}]{RESET}\n"
                )
                messages[:] = [
                    {
                        "role": "user",
                        "content": f"[Conversation summary] {summary_text}",
                    },
                    {"role": "assistant", "content": "Understood, continuing."},
                ] + messages[-4:]
            except Exception as e:
                print(f"{RED}Summarize error: {e}{RESET}\n")
</details>

## Setup
(Same as Exp-4)
* Bijaz agent (4 tools: bash, read, write, edit)
* Model: Claude Sonnet 4.5
* Diagnostic logging enabled (log_context_breakdown before every API call)
* TOKEN_STRATEGY flag switches between techniques 

## Task used
“Find top 5 Lesswrong articles and tag them according to the topic/theme”

# Quantitative Results
| Metric | Baseline | Truncate | Mask | Summarize |
|---|---|---|---|---|
| Total input tokens | 24,717 | 28,900 | 12,963 | 12,777 |
| Total output tokens | 3,104 | 4,078 | 2,368 | 3,112 |
| Total cost | $0.12 | $0.15 | $0.07 | $0.09 |
| API calls | 6 | 8 | 4 | 4 |
| Time taken | 58s | 1 min 12s | 41s | 49s |
| Task completed? | Yes | Yes | Yes | Yes |
| Correct output? | Yes | Kinda | Yes | Yes |
| Context errors? | No | Yes | No | No |
| Correct summary? | Yes | No | Yes | No |
| Biggest msg (final call) | msg[1] — full Python script (~1,964 tokens) | msg[1] (~1689 tokens) | msg[1] (~1581 tokens) | msg[1] (~1546 tokens) |
| Fixed % (final call) | 11% | 13% | 15% | 16% |
| Variable % (final call) | 89% | 87% | 85% | 84% |

## What each technique run actually did

### Baseline (6 calls)
1. wrote python script
2. ran script → ModuleNotFoundError
3. pip install requests
4. ran script again → success
5. cat JSON file (unnecessary — data already printed in call 4)
6. wrote final summary

the requests error added 2 extra calls. the unnecessary cat on call 5 added ~1,036 tokens of duplicate data to history.

### Truncate (8 calls)
1. wrote python script
2. ran script → got wrong results (recent articles with low 
   scores, not all-time top)
3. read JSON file to check
4. noticed wrong results, edited the script
5. edited script to sort by all-time score
6. ran script again → correct results
7. read JSON file again
8. wrote summary + markdown file

most calls of any technique. the extra calls were not caused by truncation — the script's initial query was wrong (LLM variance in code generation). bijaz noticed and self-corrected, which is good agent behavior. but it also read the JSON file twice (calls 3 and 7).

### Mask (4 calls)
1. wrote python script
2. ran script → success on first try
3. read JSON file
4. wrote final summary

cleanest run. no errors, correct query on first attempt. masking only activates after 6 messages, and this run only had 7 messages by the final call — so masking barely had anything to mask.

### Summarise (4 calls)
1. wrote python script
2. ran script → success on first try
3. read JSON file
4. summarization triggered before call 4 
   [summarize: +323 in / +1000 out | $0.016], then final summary

also 4 calls. summarization fired once, costing 323 input + 1,000 output tokens ($0.016 extra). despite this overhead, total cost was still lower than baseline because fewer calls.

## Observations
### Truncation has a design flaw
  * Truncation only caps tool_result content — what tools return. 
  * The biggest message in every run was msg[1], the assistant's write_file tool_use block containing the full python script. Truncation never touches assistant messages. 
  * The single largest source of token waste (identified in experiment 4) was completely unaffected by this technique.

### Masking didn't activate meaningfully
  * The sliding window keeps the last 6 messages. 
  * With only 4 API calls and 7 messages in the mask run, almost nothing got masked. 
  * The token savings came from the run having fewer calls (LLM variance), not from masking. 
  * A longer task with 15+ messages would be needed to see masking's real effect.

### Summarisation triggered once
  * One extra API call, 323 tokens in, 1,000 tokens out, $0.016. 
  * It compressed the conversation before the final call. 
  * With only 4 calls total, there wasn't much to compress. 
  * Like masking, the real test would be a longer task.

### Two runs hallucinated the final summary
  * Truncate and summarize both produced final summaries listing articles that weren't in the actual JSON data — "A LessWrong Crypto Autopsy", "My research methodology", "The Waluigi Effect", "Simulators". 
  * The JSON output was correct in both cases. 
  * Claude appears to have drawn from training data about popular LessWrong posts rather than from the actual data in context.

Baseline and Mask produced correct summaries matching the JSON. This is LLM variance, not a strategy effect — but it is worth noting that the two runs with more processing (truncate with 8 calls, summarize with compaction) were the ones that 
hallucinated.

### The biggest variable was LLM variance, not the technique
  * Baseline hit a requests error. Truncate wrote a bad query. Mask and Summarise had clean runs. 
  * These differences in code quality and error paths dominated the results more than any token strategy. 
  * The difference between 4 calls and 8 calls matters more than whether old messages were masked or summarised.

## What this means
  * This task was too short and too variable across runs to definitively compare the techniques. 
  * The token differences are dominated by whether bijaz needed 4 or 8 API calls, not by the strategies themselves.
  
To properly isolate the effect of each strategy, we would need:
  * A longer task with 15+ tool calls, where context accumulation becomes the dominant cost
  * Multiple runs per technique to separate LLM variance from strategy effects
  * A fix for the truncation design flaw — truncating assistant messages, not just tool results
  
The techniques themselves are sound in principle. The research supports them (the Complexity Trap paper found observation masking halved costs on SWE-bench tasks with 250 turns). Our task simply wasn't long enough to see the effect.
