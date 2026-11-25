# AI Agent Security: Multi-Step Tool Attacks (Kaggle, 2026)

A write-up of my entry to the Kaggle competition [AI Agent Security: Multi-Step Tool Attacks](https://www.kaggle.com/competitions/ai-agent-security-multi-step-tool-attacks), which finished in the top 15%.

The competition was an authorized red-teaming benchmark. Participants submitted an algorithm that talks to tool-using LLM agents in a sandbox and tries to make them perform unsafe tool calls despite a guardrail. This repository describes the approach and what I learned. The attack code and prompts are deliberately not published.

## The setup

Two open-weight models, GPT-OSS 20B and Gemma, ran as agents on a single T4 GPU with access to simulated tools (file system, web, email, HTTP, shell). A submission had to return a portfolio of fixed conversations. Each conversation was then replayed in a fresh environment against the public guardrail, and separately against a sealed private guardrail that nobody could test against.

A conversation scored when the agent completed a tool call that matched one of four unsafe patterns, weighted by severity, plus a small bonus for every distinct situation it reached. Each phase (searching for attacks, public replay, private replay) had a hard limit of 9,000 seconds per model, and anything that did not finish in time did not count.

## Approach

I treated it as a search problem with a fixed compute budget.

1. Probe. Run a small set of short probe conversations against each model to learn which kinds of request it complies with and how long each one takes to replay.
2. Search. Explore longer conversations with an adaptive controller that reads the agent's responses and decides the next step. It takes snapshots of the environment so that several endings can be tried from the same conversation prefix without replaying it.
3. Build the portfolio. Keep every conversation that scored, de-duplicate them, and order the final list by expected points per second of replay time, not by raw points. Add a smaller set of untested conversations aimed at the private guardrail.

The code was split into small Python modules (search controller, candidate archive, portfolio builder, logging), and a build script bundled them into the single file Kaggle required and checked message and length limits before every submission. A local harness ran the full pipeline against a deterministic fake agent to catch crashes and budget overruns before using a submission slot.

## What I learned

The binding constraint was replay time, not ideas. Doubling the portfolio from 400 to 800 conversations raised the score only from 26.8 to 31.0, because the hosted replay ran out of time after about 400 findings. From then on, the metric I optimized was points per replay second.

Local timings do not transfer. Hosted runs were roughly 6 to 12 times slower than my local T4. A version with speed gates calibrated in local seconds banned my best conversation family on the hosted run, and the score dropped from 37.4 to 24.4. The fix was to express every gate relative to a reference conversation measured in the same environment.

One crash can zero a whole model. In one version, a long conversation overflowed one model's 8,192-token context during replay and that model's entire row scored 0, while the other model scored 74.8. After that, every conversation family had to show clean, fast runs before it could enter the portfolio.

Public and private scores reward different things. Some findings were cheap to produce against the public guardrail but unlikely to survive the stricter private one, so part of the portfolio was always reserved for conversations designed to transfer.

## Score history (public leaderboard)

| Version | Score | Change |
|---|---|---|
| v1 | 26.77 | first portfolio, 400 conversations |
| v2 | 30.99 | 800 conversations; replay time became the ceiling |
| v3 | 28.20 | higher-severity conversations that were too slow |
| v4/v5 | 37.39 | best version; one model scored 0 after a context overflow |
| v6 | 24.40 | regression from gates calibrated in local seconds |
| v7 | 29.94 | speed-normalized gates |

Final result: top 15% of the leaderboard.

## Why the code is not here

The competition targeted sandboxed agents, but the conversation templates and the controller logic are a working method for pushing real tool-using agents past their guardrails. Publishing them would mostly help people misuse agents, so this repository contains only the write-up.
