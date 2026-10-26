# Tool-calling check for local models (Ollama)

A small, dependency-free script that tests how reliably a local language model
chooses the right action for short English and Hinglish shop-support messages.
It was built to pick a model for a WhatsApp order-enquiry agent for an Indian
online shop, but it works for any Ollama model and any similar routing task.

It needs only Python and a running [Ollama](https://ollama.com). No database,
no Docker, no extra packages.

## What it measures

Each test message is sent to the model together with fake tools (order status,
stock check, product search, pincode check, reply, escalate to manager). The
script records which tool the model chose and with what arguments, then scores
it against an answer key.

| Metric | Meaning |
|---|---|
| Right tool chosen | The model picked an acceptable tool for the message. "No lookup needed" cases count as correct only if the model replied or escalated instead of calling a lookup. |
| Right arguments | Where a tool was expected and chosen, the arguments contain the expected value (order number, pincode, product, reply type or escalation category). |
| Safe after guard | A wrong call counts as safe if simple code checks (digits-only order number, 6-digit pincode, a real escalation reason or valid category) would have blocked it, and no tool was the right answer anyway. This shows what validation in code can and cannot fix. |
| Same answer each run | With `--runs` above 1, the share of messages that got an identical answer every time. |
| Median seconds per reply | Median, because the first request loads the model into memory and is slow. |
| Tool call written as text | The model wrote something like `check_stock(product='x')` as plain text instead of making a real call. |

There are 41 test messages in six groups: order status, stock, search,
pincode, "no tool needed" (greetings, thanks, missing details, questions the
shop's tools cannot answer), adversarial, and escalation.

## Quick start

```
ollama pull llama3.1:8b
ollama pull qwen2.5:7b
python toolTesting.py --models llama3.1:8b qwen2.5:7b --variant schema
```

Run it from the folder where you saved the script. Results are printed and also
saved to `tool_check_results.json`. Python 3.9 or newer should work (not tested
on older versions).

The first request to each model can take a while because the model is loaded
into GPU memory. One dot is printed per reply so you can see progress.

## Variants

A variant is a way of asking the model. They were added one after another as
problems showed up, and you can compare any of them.

| Variant | What it does |
|---|---|
| `basic` | Five tools and a short prompt. The model has no way to simply reply, so it tends to call a tool even for "hi". |
| `improved` | Adds a `reply_to_customer` tool, stricter rules and examples written as function calls. Some models copy the example format as plain text. |
| `improved2` | Same rules, with examples described in words, a rule against stating facts the tools did not give, and an instruction never to write a call as text. |
| `enum` | The model never writes text. It picks a reply type or an escalation category from fixed lists, and the shop's software writes the wording. |
| `schema` | No tool calling. The model returns one JSON object whose shape is forced by Ollama's structured output, so it cannot write a call as text or invent a tool name. This is the default. |

`--variant both` runs `basic` and `improved`. `--variant all` runs `basic`,
`improved`, `improved2` and `enum`; it does not include `schema`, so run that
separately.

## Options

| Option | Default | Meaning |
|---|---|---|
| `--models` | `llama3.1:8b` | One or more Ollama model names |
| `--variant` | `schema` | `basic`, `improved`, `improved2`, `enum`, `schema`, `both` or `all` |
| `--key` | `strict` | Answer key. `strict` requires unknown-fact questions and change requests to go to the manager. `lenient` also accepts a plain reply. |
| `--runs` | `1` | Repeats per message |
| `--temperature` | `0.0` | Use 0 to 0.2 for tool selection. Higher values add random variation, and runs at 0.7 are not comparable with runs at 0. |
| `--timeout` | `90` | Seconds to wait for one reply before recording an error |
| `--num-ctx` | `4096` | Context window. Smaller uses less GPU memory. |
| `--max-tokens` | `300` | Cap on reply length. Stops runaway repeated output. |
| `--base` | `http://localhost:11434` | Ollama address |
| `--out` | `tool_check_results.json` | Where to save full results |

Examples:

```
# Consistency check at low randomness
python toolTesting.py --models qwen2.5:7b --variant schema --runs 3 --temperature 0.2

# Compare the older prompt styles on both models
python toolTesting.py --models llama3.1:8b qwen2.5:7b --variant both
```

## Results so far

Run on Windows with a good GPU and Ollama, temperature 0,
`strict` key, one run per message.

`schema` variant:

| | llama3.1:8b | qwen2.5:7b |
|---|---|---|
| Right tool chosen | 83% | 85% |
| Right arguments | 85% | 94% |
| Safe after guard | 93% | 95% |
| Median seconds per reply | 2.6 | 2.6 |

By group (correct out of total):

| Group | llama3.1:8b | qwen2.5:7b |
|---|---|---|
| Order status | 8/8 | 7/8 |
| Stock | 6/6 | 6/6 |
| Search | 6/6 | 6/6 |
| Pincode | 4/4 | 4/4 |
| No tool needed | 6/10 | 6/10 |
| Adversarial | 1/1 | 1/1 |
| Escalation | 3/6 | 5/6 |

Earlier variants, same settings (right tool chosen):

| Variant | llama3.1:8b | qwen2.5:7b |
|---|---|---|
| `basic` | 66% | 71% |
| `improved` | 85% | 54% |
| `improved2` | 83% | 90% |

The `improved2` figure for Qwen is flattering: several of its "correct"
no-tool answers were call-shaped text that a customer would have seen raw, so
the corrected figure is closer to 80%. A run of `enum` at temperature 0.7 was
also made, but it is not comparable with the rest.

### What the runs showed

- Both models are good at lookups that contain an explicit value (order
  numbers, pincodes, products). The difficulty is restraint: not calling a
  lookup when a detail is missing, and sending money, complaint and policy
  questions to a human.
- Models often fill a required field with junk (`"missing"`, `"ask_order_number"`)
  instead of asking for it. Code can catch these, which is why "safe after
  guard" is higher than "right tool".
- Examples written as function calls made Qwen write calls as plain text.
  Forcing a JSON shape with structured output removed that failure.
- Free-text fields from the 7 to 8B models were unreliable in Hinglish,
  including gibberish and invented shop facts. Letting the model choose a
  category and having code write the wording avoids this.
- The suggested design: the model only routes, and code extracts order numbers
  and pincodes, asks for missing details, escalates money-related keywords, and
  writes the replies.

## Limits of these results

Please read these before quoting any number.

- Small test: 41 messages, one run each. A single message is worth about 2.4
  percentage points, and differences of a few points are noise.
- The prompts were tuned against this same set of messages, so scores are
  optimistic. Test on a fresh, independently written set before relying on them.
- The argument check is a substring match. A value such as `">2087"` or
  `"E7788"` counts as correct for order number 2087 or 7788, so argument
  accuracy is overstated, especially for Llama.
- The tools are fake. Nothing here tests real database lookups or real
  customer conversations.
- Timing is for one machine and one GPU. Your speeds will differ.

## Adding your own test messages

Open the script and edit these:

- `CASES`: a list of `(group, message, acceptable_tools, expected_argument_substrings)`.
  Use `None` in the acceptable list to mean "no tool call".
- `STRICT`: messages whose acceptable answer changes under the strict key.
- `ENUM_KEY`: expected reply type or category for the `enum` and `schema` variants.
  A list of values means any of them is accepted.

## Troubleshooting

| Problem | What to try |
|---|---|
| `Cannot reach Ollama` | Make sure Ollama is running. On Windows it runs from the system tray. |
| `Not pulled yet` | Run the `ollama pull` command the script prints. |
| `ERROR ... timed out` | Check `ollama ps` while the model is loaded. The processor column should say GPU. Close GPU-heavy programs, or lower `--num-ctx`. Quit Ollama from the tray and start it again if it keeps stalling. |
| Three errors in a row | The script stops that setup. Restart Ollama and rerun. |
| Many "tool call written as text" | Try the `schema` variant. |
| Slow replies | Another program may be using GPU memory. Model names with larger sizes may not fit in 8 GB. |

## Model licenses

This repository does not include or distribute any model. Each model you
download has its own license, which you should read before use, especially for
anything beyond personal or academic experiments. As far as I know, Llama 3.1
is released under Meta's Llama community license and Qwen 2.5 7B under the
Apache 2.0 license, but check the model pages on Ollama and the original
publishers for the current terms.

## License

The code in this repository is released under the MIT License. See the
[LICENSE](LICENSE) file.
