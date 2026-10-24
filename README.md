E-Commerce Support Agent: Local LLM Tool-Calling Benchmark 

Tool-calling check for local models served by Ollama (version 2).
 
Sends short English and Hinglish shop messages to each model together with fake
tools, and counts how often the model picks the right tool with the right
arguments. Needs no database, no Docker and no extra packages.
 
Two setups ("variants") can be compared:
  basic     the first setup: five tools, a short prompt, no way to just reply
  improved  adds a reply_to_customer tool, stricter rules and a few examples
 
It also reports "safe after guard": a wrong call counts as safe when a simple
code check (digits-only order number, 6-digit pincode, real escalation reason)
would have blocked it and no tool was the right answer anyway. This shows what
validation in code can and cannot fix.
 
Usage:
    ollama pull llama3.1:8b
    ollama pull qwen2.5:7b
    python tool_calling_check.py --models llama3.1:8b qwen2.5:7b --variant both
    python tool_calling_check.py --models llama3.1:8b --runs 3 --temperature 0.7
 
Notes:
- Only the first tool call in each reply is scored.
- The examples in the improved prompt are deliberately different from the test
  messages, so the model is not just repeating an answer it was shown.
- The first request to each model loads it into the GPU and is slow, so the
  average uses the median.
