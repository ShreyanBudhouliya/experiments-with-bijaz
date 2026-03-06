## Hypothesis
a coding agent can understand and modify its own source code to add a working feature.

## What is the setup
* bijaz agent (has only 4 tools: bash, read, write and edit)
  * read more about bijaz here
* model: claude sonnet 4.5
* fresh conversation history for every single run (no context carryover)
* agent.py copied into working directory

## What prompt will we use to instruct bijaz
"Here is the source code for a coding agent called bijaz (agent.py). It currently has 4 tools: bash, read_file, write_file, and edit_file. Add a 5th tool called search that takes a query string and a directory path, recursively searches for the query in all files under that directory, and returns matching lines with filenames and line numbers. Update the tool schema, the execution handler, and the system prompt. When you're done, test the modified agent by running it and using the search tool."

## What does success look like
1. Modified agent.py runs without errors
2. Search tool appears in tool schemas
3. bijaz tests the tool itself
4. We test the tool manually and it works

## Results & Observations

### Results
| Metric | Result |
|---|---|
| Input tokens | 69,724 | 
| Output tokens | 4,470 | 
| Total cost | $0.276 |
| Time taken | 1 min 12s | 
| Tool calls | 10 (1 read, 4 edits, 3 bash tests, 1 grep verification, 1 write) | 
| Errors / retries | 0 |

### Observations
**did it read first?**
yes. first tool call was read_file on agent.py. it didn't start editing blind — it read the full file, understood the structure, then planned its changes.

**did it use edit_file or rewrite the whole file?**
edit_file for all 4 modifications. it only used write_file for a summary doc it created at the end (SEARCH_TOOL_CHANGES.md — nobody asked for this, but it made one anyway). this means it understood the existing code structure and surgically inserted changes rather than rewriting the whole file.

**did it hit all three places?**
yes — system prompt (four→five tools + added description), tool schema (new entry in the tools list), and execution handler (new elif block). it also added a standalone execute_search() function, which mirrors the existing execute_bash() pattern. it picked up on the codebase's conventions without being told.

**did it test its own work?**
yes, three times. searched for "execute_bash" (positive match), "SYSTEM_PROMPT" (second positive match), and "nonexistent_string_xyz" (no match). it tested the edge case unprompted.

**anything surprising?**
it added directory filtering (.git, __pycache__, node_modules, .venv) without being asked. it handled encoding errors with errors='ignore'. these are choices a human developer would make — skip junk directories, don't crash on binary files. nobody told it to do this.

### Manual Testing
after bijaz finished, i ran the modified agent.py myself to verify.

**test 1:** searched for all function definitions
```
You: Search for "def " in the current directory

[search] query='def ' path='.'
./agent.py:118:def execute_bash(command):
./agent.py:138:def execute_search(query, path):
./agent.py:169:def call_api(messages):

Claude: The search found 3 function definitions in agent.py:
1. execute_bash(command) at line 118
2. execute_search(query, path) at line 138
3. call_api(messages) at line 169
```

**test 2:** searched in a path that doesn't exist
```
You: Search for "hello" in /nonexistent

[search] query='hello' path='/nonexistent'
Error: path not found: /nonexistent

Claude: The search failed because the path /nonexistent does not exist.
```

both passed. the search tool works correctly and handles errors.

### Entire Output by bijaz
<details>
<summary>full output (10 tool calls, $0.276)</summary>

```
[tokens: +1275 in / +84 out | cost: $0.005 | session total: 1275 / 84 | total cost: $0.005]

Claude: I'll add a search tool to the bijaz agent. Let me start by reading the current agent.py file to understand its structure.

[read_file] agent.py
...full file contents...

[tokens: +3953 in / +817 out | cost: $0.024 | session total: 5228 / 901 | total cost: $0.029]

Claude: Now I'll update the agent.py file to add the search tool. I need to:
1. Update the SYSTEM_PROMPT to mention 5 tools instead of 4
2. Add the search tool to the tools array
3. Add the search handler in the execution section

Let me make these changes:

[edit_file] agent.py
OK: edited agent.py

[tokens: +4788 in / +595 out | cost: $0.023 | session total: 10016 / 1496 | total cost: $0.052]

[edit_file] agent.py
OK: edited agent.py

[tokens: +5401 in / +701 out | cost: $0.027 | session total: 15417 / 2197 | total cost: $0.079]

Claude: Now I need to add the search function implementation and the handler in the execution section:

[edit_file] agent.py
OK: edited agent.py

[tokens: +6120 in / +849 out | cost: $0.031 | session total: 21537 / 3046 | total cost: $0.110]

Claude: Now I need to add the handler in the tool execution section:

[edit_file] agent.py
OK: edited agent.py

[tokens: +6987 in / +108 out | cost: $0.023 | session total: 28524 / 3154 | total cost: $0.133]

Claude: Great! Now let's test the modified agent by running it and using the search tool.

[bash] python agent.py << 'EOF'
Search for the string "execute_bash" in the current directory
/quit
EOF

[search] query='execute_bash' path='.'
./agent.py:118:def execute_bash(command):
./agent.py:251:                    result = execute_bash(block.input["command"])

Claude: I found the string "execute_bash" in the file `./agent.py`:
- Line 118: Function definition `def execute_bash(command):`
- Line 251: Function call `result = execute_bash(block.input["command"])`

[tokens: +7370 in / +102 out | cost: $0.024 | session total: 35894 / 3256 | total cost: $0.157]

[bash] python agent.py << 'EOF'
Search for "SYSTEM_PROMPT" in the current directory
/quit
EOF

[search] query='SYSTEM_PROMPT' path='.'
./agent.py:17:SYSTEM_PROMPT = """You are an expert coding agent...
./agent.py:176:                system=SYSTEM_PROMPT,

[tokens: +7797 in / +101 out | cost: $0.025 | session total: 43691 / 3357 | total cost: $0.181]

[bash] python agent.py << 'EOF'
Search for "nonexistent_string_xyz" in the current directory
/quit
EOF

[search] query='nonexistent_string_xyz' path='.'
No matches found for 'nonexistent_string_xyz' in .

[tokens: +8118 in / +86 out | cost: $0.026 | session total: 51809 / 3443 | total cost: $0.207]

[bash] grep -n "def execute_search" agent.py -A 30
138:def execute_search(query, path):
139-    try:
140-        matches = []
141-        if not os.path.exists(path):
142-            return f"Error: path not found: {path}"
143-        if not os.path.isdir(path):
144-            return f"Error: path is not a directory: {path}"
145-        for root, dirs, files in os.walk(path):
146-            dirs[:] = [d for d in dirs if d not in ['.git', '__pycache__', 'node_modules', '.venv', 'venv']]
147-            for file in files:
148-                filepath = os.path.join(root, file)
149-                try:
150-                    with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
151-                        for line_num, line in enumerate(f, 1):
152-                            if query in line:
153-                                matches.append(f"{filepath}:{line_num}:{line.rstrip()}")
154-                except (PermissionError, IsADirectoryError):
155-                    continue
156-        if not matches:
157-            return f"No matches found for '{query}' in {path}"
158-        return '\n'.join(matches)
159-    except Exception as e:
160-        return f"Error: {e}"

[tokens: +8598 in / +694 out | cost: $0.036 | session total: 60407 / 4137 | total cost: $0.243]

[write_file] SEARCH_TOOL_CHANGES.md
OK: wrote SEARCH_TOOL_CHANGES.md

[tokens: +9317 in / +333 out | cost: $0.033 | session total: 69724 / 4470 | total cost: $0.276]

Claude: I successfully added a 5th tool called `search` to the bijaz agent.
```

</details>
