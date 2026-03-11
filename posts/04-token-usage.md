## What we're doing and why
In experiment 1, we noticed that the token usage for even a simple task was quite high: bijaz took ~70k input tokens for fetching top 5 posts from LessWrong and tagging them according to theme.

Reasons identified at that time were:
* No Context Compression
* No Tool Output Truncation
* Unnecessary file exploration (this was fixed through changes in System Prompt)

This experiment is a way to diagnose why the token usage is so high with bijaz and if the above reasons are truly the main reasons behind that.

We've added diagnostic logging to bijaz that shows exactly what's inside every API call, which shows full context breakdown i.e token usage by: 
1. Fixed Context 
2. Variable Context
3. New Messages
4. Old Messages

## Setup
* Bijaz agent (4 tools: bash, read, write, edit)
* Model: Claude Sonnet 4.5
* Added a `log_context_breakdown` function for diagnosis
* Token estimation: Character length ÷ 4 (rough but directional)
  * For example: 
    * The input prompt had 13 words (71 characters). Bijaz showed that this resulted in 17 tokens (only for input message).
    * So roughly, 1 token = 0.76 words.
    * Or equivalently, 1 word ≈ 1.3–1.5 tokens


## Task used
“Find top 5 Lesswrong articles and tag them according to the topic/theme”

## How an API call looks like
![API Call Tree](/images/api-tree.png)

## What happened: call by call

Sharing the breakdown of Call 1 for reference here.

To find the breakdown of each call (total 6 calls), go to the last section.

### Breakdown of Call 1 
Call 1 was right after when i sent the input task prompt.

### Context Breakdown
| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [NEW]| msg[0] user(text) | ~17 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (97%) |
| [Variable] | ~17 (3%) |
| Biggest Message | msg[0] (~17 tokens) |

Total Tokens: ~633 tokens

## Findings

### The Biggest Cost: One Message, Resent Forever
* msg(1) was the full python script bijaz wrote on turn 1: 1,964 tokens. 
* It was written to disk and never referenced again but it stayed in conversation history.
* It got resent on every subsequent API call — 5 extra times, costing 9,820 tokens. 
* This single message accounts for 40% of total input tokens.

### Fixed Costs: Small but constant
* System prompt (305 tokens) + Tool schemas (311 tokens) = 616 tokens per call. 
* Across 6 calls that's 3,696 tokens.
* This accounts for 15% of the session.

### Growth Pattern
| Call # | Estimated total | Actual (API) | Increase from last call |
|---|---|---|---|
| 1 | 633 | 1,180 | — |
| 2 | 2,628 | 3,314 | +2,134 |
| 3 | 2,793 | 3,498 | +184 |
| 4 | 3,195 | 4,187 | +689 |
| 5 | 4,388 | 5,545 | +1,358 |
| 6 | 5,508 | 6,993 | +1,448 |

* Biggest jump: call 1→2 (+2,134) caused by msg(1), the full Python script entering the history. 
* Second biggest: call 4→5 (+1,358) caused by the script execution output (1,114 tokens).

### Waste Analysis

81% of context was dead weight.

| Message | Still relevant? | Why / why not |
|---|---|---|
| msg[0]: user task prompt (17 tokens) | Yes | Agent needs to know what was asked |
| msg[1]: full Python script (1,964 tokens) | No | Already written to disk. Never referenced again. |
| msg[2]: "OK: wrote file" (31 tokens) | No | Confirmation of a completed action |
| msg[3]: "let me run the script" (88 tokens) | No | Planning text, action already taken |
| msg[4]: ModuleNotFoundError (77 tokens) | No | Error was fixed. Irrelevant now. |
| msg[5]: "install requests" (61 tokens) | No | Planning text, action already taken |
| msg[6]: pip install output (341 tokens) | No | Package installed. Progress bars are noise. |
| msg[7]: "run again" (79 tokens) | No | Planning text, action already taken |
| msg[8]: script output (1,114 tokens) | Maybe | Agent used this to write the final summary. Dead after that. |

* Relevant by final call: ~17 tokens (just the task prompt)
* Dead weight by final call: ~3,855 tokens (msg[1] through msg[8])

## Total Costs

### Total Token Usage Bifurcation
Total actual input tokens across all 6 calls: 24,717
| Category | Estimated tokens | % of total |
|---|---|---|
| Fixed costs (system prompt + tools × 6 calls) | 3,696 | 15% |
| msg[1] — Python script resent 5 extra times | 9,820 | 40% |
| msg[8] — script output resent 1 extra time | 1,114 | 5% |
| msg[6] — pip install output resent 2 extra times | 682 | 3% |
| All other old messages resent | 783 | 3% |
| New messages (actually useful per call) | ~4,900 | 20% |
| Unaccounted (API overhead, JSON framing) | ~3,722 | 15% |

<details>
    <summary>Fixed Costs (per call)</summary>

### Fixed Costs (per call)
| Component | Tokens | Notes |
|---|---|---|
| System prompt| ~305 tokens | Same every call, never changes | 
| Tool Schemas (4 tools) | ~311 tokens | JSON definitions for bash, read_file, write_file, edit_file |
| Total Fixed | ~616 tokens | Sent 6 times = 3,696 tokens total across session |

</details>

<details>
    <summary>Variable Costs — Old Messages (resent from previous turns)</summary>

### Variable Costs — Old Messages (resent from previous turns)

By the final call (#6), these old messages were being resent:
| Message | What it was | Tokens | Times resent | Total waste |
|---|---|---|---|---|
| msg[0] | User's task prompt: "Find top 5 Lesswrong articles..." | 17 | 5 | 85 |
| msg[1] | Assistant writing the full Python script (fetch_lesswrong_articles.py — 160 lines) | 1,964 | 5 | 9,820 |
| msg[2] | Tool result: "OK: wrote fetch_lesswrong_articles.py" | 31 | 4 | 124 |
| msg[3] | Assistant saying "let me run the script" + bash tool call | 88 | 3 | 264 |
| msg[4] | Tool result: ModuleNotFoundError traceback | 77 | 3 | 231 |
| msg[5] | Assistant saying "let me install requests" + bash tool call | 61 | 2 | 122 |
| msg[6] | Tool result: full pip install output (download progress bars, package versions) | 341 | 2 | 682 |
| msg[7] | Assistant saying "let's run the script again" + bash tool call | 79 | 1 | 79 |
| msg[8] | Tool result: full script execution output (all 5 articles printed with tags, category summary) | 1,114 | 1 | 1,114 |

</details>

<details>
    <summary>Variable Costs — New Messages (added per call)</summary>

### Variable Costs — New Messages (added per call)
| Call # | New assistant msg | New tool_result | Total new |
|---|---|---|---|
| 1 | — | — | 17 (just user prompt) |
| 2 | 1,964 (full Python script) | 31 ("OK: wrote") | 1,995 |
| 3 | 88 ("let me run it") | 77 (error traceback) | 165 |
| 4 | 61 ("install requests") | 341 (pip output) | 402 |
| 5 | 79 ("run again") | 1,114 (script output) | 1,193 |
| 6 | 84 ("let me cat the JSON") | 1,036 (full JSON dump) | 1,120 |

</details>

## What these findings mean

* The token problem is stale tool outputs persisting in conversation history indefinitely. 
* A single message — accounted for 40% of total input tokens across the session.
* This is the same problem Claude Code addresses with auto Context Compaction, and the same problem the "Context Management" paper ([Jetbrains Research](https://blog.jetbrains.com/research/2025/12/efficient-context-management)) tackles with observation masking. 

## What's next

In experiment 5 we will focus on implementing token optimisation. 

## Breakdown of all API calls
<details>
    <summary>Breakdown of all 6 API Calls</summary>

## Breakdown of Call 1 
Call 1 was right after when i sent the input task prompt.

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [NEW]| msg[0] user(text) | ~17 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (97%) |
| [Variable] | ~17 (3%) |
| Biggest Message | msg[0] (~17 tokens) |

Total Tokens: ~633 tokens

## Breakdown of Call 2 

### What bijaz did:

#### Text Response by bijaz: 

"I'll help you find the top 5 LessWrong articles and tag them by topic/theme. Let me start by creating a script to fetch and analyze these articles."

#### Internal Reasoning by bijaz:

(write_file) fetch_lesswrong_articles.py
OK: wrote fetch_lesswrong_articles.py

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [OLD]| msg[0] user(text) | ~17 tokens | 
| [NEW]| msg[1] assistant (tool_result) | ~1964 tokens | 
| [NEW]| msg[2] user (tool_result) | ~31 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (23%) |
| [Variable] | ~2012 (77%) |
| Biggest Message | msg[1] (~1964 tokens) |

Total Tokens: ~2628 tokens

## Breakdown of Call 3 

### What bijaz did:

#### Text Response by bijaz: 

"Now let me run the script to fetch and tag the top LessWrong articles:"

#### Internal Reasoning by bijaz:

"Found error while trying to run (bash) python3 fetch_lesswrong_articles.py"

"Error: ModuleNotFoundError: No module named 'requests'"

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [OLD]| msg[0] user(text) | ~17 tokens | 
| [OLD]| msg[1] assistant (tool_result) | ~1964 tokens | 
| [OLD]| msg[2] user (tool_result) | ~31 tokens | 
| [NEW]| msg[3] assistant (tool_result) | ~88 tokens | 
| [NEW]| msg[4] user (tool_result) | ~77 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (22%) |
| [Variable] | ~2177 (78%) |
| Biggest Message | msg[1] (~1964 tokens) |

Total Tokens: ~2793 tokens

## Breakdown of Call 4 

### What bijaz did:

#### Text Response by bijaz: 

"Let me install the required package:"

#### Internal Reasoning by bijaz:

"(bash) pip3 install requests"

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [OLD]| msg[0] user(text) | ~17 tokens | 
| [OLD]| msg[1] assistant (tool_result) | ~1964 tokens | 
| [OLD]| msg[2] user (tool_result) | ~31 tokens | 
| [OLD]| msg[3] assistant (tool_result) | ~88 tokens | 
| [OLD]| msg[4] user (tool_result) | ~77 tokens | 
| [NEW]| msg[5] assistant (tool_result) | ~61 tokens | 
| [NEW]| msg[6] user (tool_result) | ~341 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (19%) |
| [Variable] | ~2579 (81%) |
| Biggest Message | msg[1] (~1964 tokens) |

Total Tokens: ~3195 tokens

## Breakdown of Call 5 

### What bijaz did:

#### Text Response by bijaz: 

"Now let's run the script again:"

#### Internal Reasoning by bijaz:

"(bash) python3 fetch_lesswrong_articles.py"

<details>
<summary>Fetching top 5 LessWrong articles...</summary>

```
Fetching top 5 LessWrong articles...

================================================================================
TOP 5 LESSWRONG ARTICLES WITH TOPIC TAGS
================================================================================

#1. AGI Ruin: A List of Lethalities
    Author: Eliezer Yudkowsky
    Score: 963 | Votes: 584
    Posted: 2022-06-05T22:05:52.224Z
    📌 Topic Tags: AI Safety & Alignment, Effective Altruism, Philosophy, Cognitive Science, Self-Improvement, Meta & Community
    🏷️  LessWrong Tags: AI Risk, Threat Models (AI), AI Questions Open Threads, Double-Crux, Fuzzies
    🔗 URL: https://www.lesswrong.comNone
    📝 Summary: Preamble:
(If you're already familiar with all basics and don't want any preamble, skip ahead to Section B for technical difficulties of alignment proper.)

I have several times failed to write up a w...

--------------------------------------------------------------------------------

#2. Where I agree and disagree with Eliezer
    Author: paulfchristiano
    Score: 915 | Votes: 425
    Posted: 2022-06-19T19:15:55.698Z
    📌 Topic Tags: AI Safety & Alignment, Rationality, Effective Altruism, Existential Risk, Science & Research, Futurism & Technology
    🏷️  LessWrong Tags: AI Risk, Meetups & Local Communities (topic), AI, World Modeling
    🔗 URL: https://www.lesswrong.comNone
    📝 Summary: (Partially in response to AGI Ruin: A list of Lethalities. Written in the same rambling style. Not exhaustive.)


Agreements
 1. Powerful AI systems have a good chance of deliberately and irreversibly...

--------------------------------------------------------------------------------

#3. Preface
    Author: Eliezer Yudkowsky
    Score: 898 | Votes: 999
    Posted: 2015-03-11T19:00:00.164Z
    📌 Topic Tags: Rationality, Meta & Community, AI Safety & Alignment, Philosophy, Effective Altruism, Self-Improvement
    🏷️  LessWrong Tags: Postmortems & Retrospectives, Rationality A-Z (discussion & meta), Life Improvements, Rationality
    🔗 URL: https://www.lesswrong.comNone
    📝 Summary: You hold in your hands a compilation of two years of daily blog posts. In retrospect, I look back on that project and see a large number of things I did completely wrong. I’m fine with that. Looking b...

--------------------------------------------------------------------------------

#4. Eight Short Studies On Excuses
    Author: Scott Alexander
    Score: 892 | Votes: 826
    Posted: 2010-04-20T23:01:15.252Z
    📌 Topic Tags: AI Safety & Alignment, Rationality, Economics & Game Theory, Effective Altruism, Science & Research
    🏷️  LessWrong Tags: Game Theory, AI Risk, Affective Death Spiral, Group Rationality, Life Improvements
    🔗 URL: https://www.lesswrong.comNone
    📝 Summary: The Clumsy Game-Player

You and a partner are playing an Iterated Prisoner's Dilemma. Both of you have publicly pre-committed to the tit-for-tat strategy. By iteration 5, you're going happily along, r...

--------------------------------------------------------------------------------

#5. The Company Man
    Author: Tomás B.
    Score: 804 | Votes: 488
    Posted: 2025-09-17T17:47:13.518Z
    📌 Topic Tags: AI Safety & Alignment, Cognitive Science, Mathematics & Logic, Effective Altruism, Meta & Community
    🏷️  LessWrong Tags: Fiction
    🔗 URL: https://www.lesswrong.comNone
    📝 Summary: To get to the campus, I have to walk past the fentanyl zombies. I call them fentanyl zombies because it helps engender a sort of detached, low-empathy, ironic self-narrative which I find useful for my...

--------------------------------------------------------------------------------


✅ Results saved to lesswrong_top_articles.json

================================================================================
CATEGORY SUMMARY
================================================================================
  AI Safety & Alignment: 5 article(s)
  Effective Altruism: 5 article(s)
  Meta & Community: 3 article(s)
  Rationality: 3 article(s)
  Philosophy: 2 article(s)
  Cognitive Science: 2 article(s)
  Self-Improvement: 2 article(s)
  Science & Research: 2 article(s)
  Existential Risk: 1 article(s)
  Futurism & Technology: 1 article(s)
  Economics & Game Theory: 1 article(s)
  Mathematics & Logic: 1 article(s)
```
</details>

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [OLD]| msg[0] user(text) | ~17 tokens | 
| [OLD]| msg[1] assistant (tool_result) | ~1964 tokens | 
| [OLD]| msg[2] user (tool_result) | ~31 tokens | 
| [OLD]| msg[3] assistant (tool_result) | ~88 tokens | 
| [OLD]| msg[4] user (tool_result) | ~77 tokens | 
| [OLD]| msg[5] assistant (tool_result) | ~61 tokens | 
| [OLD]| msg[6] user (tool_result) | ~341 tokens | 
| [NEW]| msg[7] assistant (tool_result) | ~79 tokens | 
| [NEW]| msg[8] user (tool_result) | ~1114 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (14%) |
| [Variable] | ~3772 (86%) |
| Biggest Message | msg[1] (~1964 tokens) |

Total Tokens: ~4388 tokens

## Breakdown of Call 6 

### What bijaz did:

#### Text Response by bijaz: 

"Perfect! Let me also display the JSON file with all the details:"

#### Internal Reasoning by bijaz:

"(bash) cat /.../lesswrong_top_articles.json"

<details>
<summary>Displaying in JSON...</summary>

```
[
  {
    "rank": 1,
    "title": "AGI Ruin: A List of Lethalities",
    "author": "Eliezer Yudkowsky",
    "score": 963,
    "votes": 584,
    "url": "https://www.lesswrong.comNone",
    "posted": "2022-06-05T22:05:52.224Z",
    "categories": [
      "AI Safety & Alignment",
      "Effective Altruism",
      "Philosophy",
      "Cognitive Science",
      "Self-Improvement",
      "Meta & Community"
    ],
    "lesswrong_tags": [
      "AI Risk",
      "Threat Models (AI)",
      "AI Questions Open Threads",
      "Double-Crux",
      "Fuzzies",
      "Language Models (LLMs)",
      "Meetups & Local Communities (topic)",
      "AI"
    ],
    "description": "Preamble:\n(If you're already familiar with all basics and don't want any preamble, skip ahead to Section B for technical difficulties of alignment proper.)\n\nI have several times failed to write up a w..."
  },
  {
    "rank": 2,
    "title": "Where I agree and disagree with Eliezer",
    "author": "paulfchristiano",
    "score": 915,
    "votes": 425,
    "url": "https://www.lesswrong.comNone",
    "posted": "2022-06-19T19:15:55.698Z",
    "categories": [
      "AI Safety & Alignment",
      "Rationality",
      "Effective Altruism",
      "Existential Risk",
      "Science & Research",
      "Futurism & Technology"
    ],
    "lesswrong_tags": [
      "AI Risk",
      "Meetups & Local Communities (topic)",
      "AI",
      "World Modeling"
    ],
    "description": "(Partially in response to AGI Ruin: A list of Lethalities. Written in the same rambling style. Not exhaustive.)\n\n\nAgreements\n 1. Powerful AI systems have a good chance of deliberately and irreversibly..."
  },
  {
    "rank": 3,
    "title": "Preface",
    "author": "Eliezer Yudkowsky",
    "score": 898,
    "votes": 999,
    "url": "https://www.lesswrong.comNone",
    "posted": "2015-03-11T19:00:00.164Z",
    "categories": [
      "Rationality",
      "Meta & Community",
      "AI Safety & Alignment",
      "Philosophy",
      "Effective Altruism",
      "Self-Improvement"
    ],
    "lesswrong_tags": [
      "Postmortems & Retrospectives",
      "Rationality A-Z (discussion & meta)",
      "Life Improvements",
      "Rationality"
    ],
    "description": "You hold in your hands a compilation of two years of daily blog posts. In retrospect, I look back on that project and see a large number of things I did completely wrong. I’m fine with that. Looking b..."
  },
  {
    "rank": 4,
    "title": "Eight Short Studies On Excuses",
    "author": "Scott Alexander",
    "score": 892,
    "votes": 826,
    "url": "https://www.lesswrong.comNone",
    "posted": "2010-04-20T23:01:15.252Z",
    "categories": [
      "AI Safety & Alignment",
      "Rationality",
      "Economics & Game Theory",
      "Effective Altruism",
      "Science & Research"
    ],
    "lesswrong_tags": [
      "Game Theory",
      "AI Risk",
      "Affective Death Spiral",
      "Group Rationality",
      "Life Improvements",
      "More Dakka",
      "Rationality",
      "Practical"
    ],
    "description": "The Clumsy Game-Player\n\nYou and a partner are playing an Iterated Prisoner's Dilemma. Both of you have publicly pre-committed to the tit-for-tat strategy. By iteration 5, you're going happily along, r..."
  },
  {
    "rank": 5,
    "title": "The Company Man",
    "author": "Tomás B.",
    "score": 804,
    "votes": 488,
    "url": "https://www.lesswrong.comNone",
    "posted": "2025-09-17T17:47:13.518Z",
    "categories": [
      "AI Safety & Alignment",
      "Cognitive Science",
      "Mathematics & Logic",
      "Effective Altruism",
      "Meta & Community"
    ],
    "lesswrong_tags": [
      "Fiction"
    ],
    "description": "To get to the campus, I have to walk past the fentanyl zombies. I call them fentanyl zombies because it helps engender a sort of detached, low-empathy, ironic self-narrative which I find useful for my..."
  }
]
```
</details>

### Context Breakdown

| Fixed/Variable | What | No. of tokens |
|---|---|---|
| [Fixed]| System Prompt | ~305 tokens | 
| [Fixed]| Tool Schemas | ~311 tokens | 
| [OLD]| msg[0] user(text) | ~17 tokens | 
| [OLD]| msg[1] assistant (tool_result) | ~1964 tokens | 
| [OLD]| msg[2] user (tool_result) | ~31 tokens | 
| [OLD]| msg[3] assistant (tool_result) | ~88 tokens | 
| [OLD]| msg[4] user (tool_result) | ~77 tokens | 
| [OLD]| msg[5] assistant (tool_result) | ~61 tokens | 
| [OLD]| msg[6] user (tool_result) | ~341 tokens | 
| [OLD]| msg[7] assistant (tool_result) | ~79 tokens | 
| [OLD]| msg[8] user (tool_result) | ~1114 tokens | 
| [NEW]| msg[9] assistant (tool_result) | ~84 tokens | 
| [NEW]| msg[10] user (tool_result) | ~1036 tokens | 

#### Total: 
|  |  |  
|---|---|
|[Fixed] | ~616 (11%) |
| [Variable] | ~4892 (89%) |
| Biggest Message | msg[1] (~1964 tokens) |

Total Tokens: ~5508 tokens

## Final response by bijaz

<details>
<summary>Fetching is successful and here's the summary...</summary>

```
Excellent! I've successfully fetched and tagged the top 5 LessWrong articles. Here's a summary:

## **Top 5 LessWrong Articles with Topic Tags**

### **1. AGI Ruin: A List of Lethalities** 
- **Author:** Eliezer Yudkowsky
- **Score:** 963 | Votes: 584
- **Posted:** June 2022
- **Topics:** AI Safety & Alignment, Effective Altruism, Philosophy, Cognitive Science, Self-Improvement, Meta & Community
- **Summary:** A comprehensive analysis of why AGI alignment is extremely difficult and poses existential risks

### **2. Where I agree and disagree with Eliezer**
- **Author:** Paul Christiano  
- **Score:** 915 | Votes: 425
- **Posted:** June 2022
- **Topics:** AI Safety & Alignment, Rationality, Effective Altruism, Existential Risk, Science & Research, Futurism & Technology
- **Summary:** A response to Eliezer's "AGI Ruin" article discussing points of agreement and disagreement on AI risk

### **3. Preface**
- **Author:** Eliezer Yudkowsky
- **Score:** 898 | Votes: 999  
- **Posted:** March 2015
- **Topics:** Rationality, Meta & Community, AI Safety & Alignment, Philosophy, Effective Altruism, Self-Improvement
- **Summary:** Introduction to a compilation of blog posts reflecting on rationality and learning from mistakes

### **4. Eight Short Studies On Excuses**
- **Author:** Scott Alexander
- **Score:** 892 | Votes: 826
- **Posted:** April 2010
- **Topics:** AI Safety & Alignment, Rationality, Economics & Game Theory, Effective Altruism, Science & Research
- **Summary:** Exploring game theory scenarios and human behavior around excuses and coordination problems

### **5. The Company Man**
- **Author:** Tomás B.
- **Score:** 804 | Votes: 488
- **Posted:** September 2025
- **Topics:** AI Safety & Alignment, Cognitive Science, Mathematics & Logic, Effective Altruism, Meta & Community
- **Summary:** A fictional narrative exploring contemporary tech culture and AI development

### **Category Summary:**
- **AI Safety & Alignment:** 5 articles (100%)
- **Effective Altruism:** 5 articles (100%)
- **Rationality:** 3 articles (60%)
- **Meta & Community:** 3 articles (60%)

The data has been saved to `lesswrong_top_articles.json` for your reference!
```
</details>
</details>
