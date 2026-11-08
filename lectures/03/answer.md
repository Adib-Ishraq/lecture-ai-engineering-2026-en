# Lecture 03 Answers

## Exercises

- Differences between the Lecture 01 (bare LLM) and agent answers:
  - Lecture 01 (`app.ask`): `gemini-3.5-flash-lite` said it had no information about the AIE2026 course, asked me to share the syllabus, and did not list the levels.
  - Agent (`app.agent`) on "What are the “3 levels” in the AIE2026 course homework?": it searched the glossary, listed Basic / Implementation / Advanced, said any one level is enough, and ended with the lecture number (Lecture 1).
  - Compared with `data/glossary.json` ("3 homework levels"): matches. The glossary lists the same three levels ("review lecture content", "write your own code", "research and experimentation"), says any one completes the homework, and says it is not graded as a rule. The agent's answer agreed on all of this.
- Tool calls shown by `--trace`:
  - "3 levels": `search_glossary({'query': '3 levels'})`
  - "What is the difference between LoRA and QLoRA?": `search_glossary({'query': 'LoRA'})` and `search_glossary({'query': 'QLoRA'})`
- Number of LLM calls for one question (`requests`):
  - "3 levels": requests=2, tool_calls=1
  - LoRA vs QLoRA: requests=2, tool_calls=2. Both searches were issued in one model response, so they count as one request, plus one for the final answer.
- Tool added and the description in its docstring:
  

## Homework

- Level attempted: Basic
- What I did:
  - Changed one place at a time, ran the same question with `--trace`, and reverted with `git checkout` between experiments.
  - A: deleted the "Always check the glossary with search_glossary..." line from INSTRUCTIONS (`app/agent.py`).
  - B: changed the `search_glossary` docstring in `app/tools.py` from "Put only one term" to "Put several terms".
  - Q1 = "What is the difference between LoRA and QLoRA?"; Q2 = "What are the “3 levels” in the AIE2026 course homework?"
  - Runs: baseline Q1 x2, baseline Q2 x1, A Q1 x1, A Q2 x1, B Q1 x1.
- Results:

  | Case | Q | Tool calls | requests / tool_calls | input tokens | cost (USD) | Note |
  |---|---|---|---|---|---|---|
  | Baseline (run 1) | Q1 | search_glossary('LoRA'), ('QLoRA') | 2 / 2 | 1129 | 0.00084 | ends "(Lecture 02)" |
  | Baseline (run 2) | Q1 | same | 2 / 2 | 1129 | 0.00100 | same calls, different wording |
  | Baseline | Q2 | search_glossary('3 levels') | 2 / 1 | 741 | 0.00060 | ends "Lecture 1" |
  | A | Q1 | same two searches | 2 / 2 | 1087 | 0.00088 | still searched |
  | A | Q2 | get_lecture_summary(1..8) one per request, then search_glossary('level') | 10 / 9 | 17101 | 0.00578 | same answer content, ends "(Lecture 01)" |
  | B | Q1 | search_glossary('LoRA QLoRA') | 2 / 1 | 834 | 0.00076 | one merged call |

- Observations and questions:
  - A (INSTRUCTIONS): removing the "always search" line did not stop the search on Q1. On Q2 it changed the strategy: instead of one glossary search, the agent called get_lecture_summary for lectures 1-8 one by one (4, 6, 7, 8 returned "not available yet"), then fell back to search_glossary('level'). The answer content was the same, but requests went 2 -> 10, input tokens 741 -> 17101 (~23x), and cost rose ~9.6x. Hypothesis, not tested: the only tool rule left was "use get_lecture_summary when asked about the content of a session", and Q2 names no lecture, so the model scanned all of them.
  - A also shortened the system prompt: Q1 input tokens fell 1129 -> 1087, consistent with one removed line.
  - B (docstring): the docstring controlled the argument shape. Two searches became one merged query. tool_calls fell 2 -> 1 but requests stayed 2, because requests counts LLM round trips, not tool calls. Input tokens fell 1129 -> 834 and cost was ~9-24% lower. I called `search_glossary('LoRA QLoRA')` directly and it returned both the LoRA and QLoRA entries, so the merged query lost nothing.
  - Variance: Q1 baseline had identical tool calls in 2 runs but different wording, and the lecture marker format varied ("(Lecture 02)", "Lecture 02", "Lecture 1", "*(Lecture 01)*"). Q2 baseline, A and B are single runs, so the 9-call sweep in A is one observation, not a rate.
  - Verified myself: compared the Q2 answers (baseline and A) against the "3 homework levels" entry in `data/glossary.json`, including the "not graded" claim; checked B's merged query output directly. Not verified: why A triggered the sweep (hypothesis above).
  - Question: how often does the agent sweep all lecture summaries without the glossary line (over N runs), and is it driven by the question not naming a lecture?
  - AI tools: Claude helped plan the experiments and draft this file. I ran all commands and checked the results.