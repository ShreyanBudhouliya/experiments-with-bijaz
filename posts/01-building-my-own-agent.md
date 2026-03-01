this entire website was built by bijaz, a coding agent i built in less than a day.

it's a pretty simple and stripped down agent, consisting of ~250 lines of python. the project was a inspired by mario's pi-agent(https://mariozechner.at/posts/2025-11-30-pi-coding-agent/).

currently, it only has 4 tools:
1. bash
2. read_file
3. write_file
4. edit_file

the entire thing was almost entirely built by claude code, with some edge cases and errors handled by me.

the build process was divided into modules, to make it easier for me to understand what claude code was doing at each step instead of just one-shotting it:
1. module 1: hello claude
  * just called the claude api and got a response.
2. module 2: conversation loop
  * added multi-turn functionality: you send a message, claude responds, you reply and so on.
  * main thing to test at this step was if claude is remembering context correctly or not
3. module 3: tool definitions & bash tool executor 
  * defined the four tools (bash, read_file, write_file, edit_file) as json schemas.
  * test the above defined tools with simple commands like "list the files in this directory."
4. module 4: file tool executors 
  * add read_file, write_file, edit_file execution, now bijaz could read, write, and modify files.
5. module 5: the agent loop
  * wire everything together: bijaz could now make multiple tool calls in sequence without me intervening.
  * added a system prompt at this point too. following mario's philosphy on keeping everything as bare bones, this system prompt was also pretty simple.
6. module 6: polish things and fix bugs
  * added simple commands like: 
    * /clear: clears context and conversation history gets reset
    * /quit: to quit the conversation (added in module 2 only)
    * ctrl+c: leads to a clean exit during bijaz's execution
  * added token & cost display: after any interaction, bijaz shows you how many tokens and how much cost consumed.
  * added color functionality: so that i could differ between my response and bijaz's reply.

the biggest issues we face right now are:
1. token usage is high
  *  for a simple api-fetching task, bijaz used ~70k input tokens. for the same task,
      * claude code would've used roughly ~25-30k tokens
      * pi-agent would've used roughly ~40-50k tokens
  * main reasons we've identified for this high usage are:
    1. no context compression: it resends the full conversation history on every call, for even a simple task involving multiple calls that's basically quadratic growth in token usage.
    2. unnecessary file exploration: before starting the task, it was finding and reading all the files in current directory. 
    3. tool output truncation: large tool outputs (like api responses) get stored in conversation history at full length and repeat on every subsequent api call.
2. it retries without reasoning
  * when bijaz hit errors during a task (like a 400 from an api), it would immediately try a different approach without explaining why the previous one failed. 
3. tui improvements needed 
  * right now bijaz dumps everything to the terminal: full file contents on every read_file, complete api responses, all internal reasoning. this becomes unreadable after a certain point. 

issues of 'unnecessary file exploration' and 'it retries without reasoning' were fixed through making changes in the system prompt.

bijaz is ~5% of claude code's codebase but covers the core agent loop. the other 95% is efficiency, ux, and tool sophistication.

i'm excited to try out different things with bijaz and improve it as we go ahead.
