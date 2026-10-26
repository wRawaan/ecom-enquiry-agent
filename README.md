Tool-calling check for local models served by Ollama (version 2).

Sends short English and Hinglish shop messages to each model together with fake
tools, and counts how often the model picks the right tool with the right
arguments. Needs no database, no Docker and no extra packages.

Two setups ("variants") can be compared:
  basic     the first setup: five tools, a short prompt, no way to just reply
  improved  adds a reply_to_customer tool, stricter rules and a few examples
  improved2 same, with examples in words and a rule against guessing shop facts
  enum      the model never writes text: it picks a reply type or an escalation category
            from fixed lists, and the shop's software writes the wording
  schema    no tool calling at all: the model returns one JSON object whose shape is forced
            by Ollama (structured output), so it cannot write a call as text or invent a tool

It also reports "safe after guard": a wrong call counts as safe when a simple
code check (digits-only order number, 6-digit pincode, real escalation reason)
would have blocked it and no tool was the right answer anyway. This shows what
validation in code can and cannot fix.

Usage:
    ollama pull llama3.1:8b
    ollama pull qwen2.5:7b
    python toolTesting.py --models llama3.1:8b qwen2.5:7b --variant both
    python toolTesting.py --models llama3.1:8b --runs 3 --temperature 0.7
    python toolTesting.py --models llama3.1:8b qwen2.5:7b --variant schema

Notes:
- Only the first tool call in each reply is scored.
- The examples in the improved prompt are deliberately different from the test
  messages, so the model is not just repeating an answer it was shown.
- The first request to each model loads it into the GPU and is slow, so the
  average uses the median.

  Summary: 

llama3.1:8b [schema]
  Right tool chosen:     83%
  Right arguments:       85%  (where a tool was expected)
  Safe after guard:      93%
  Median seconds/reply:  2.6
  By category: order 8/8  stock 6/6  search 6/6  pincode 4/4  no_tool 6/10  adversarial 1/1  escalate 3/6

qwen2.5:7b [schema]
  Right tool chosen:     85%
  Right arguments:       94%  (where a tool was expected)
  Safe after guard:      95%
  Median seconds/reply:  2.6
  By category: order 7/8  stock 6/6  search 6/6  pincode 4/4  no_tool 6/10  adversarial 1/1  escalate 5/6

                        llama3.1:8b [schema]  qwen2.5:7b [schema]
Right tool              83%                   85%
Right arguments         85%                   94%
Safe after guard        93%                   95%
Median seconds/reply    2.6                   2.6
