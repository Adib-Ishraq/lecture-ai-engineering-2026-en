## Homework

- Level attempted: Implementation

- Settings or data changed:
  - Replaced the course FAQ/GROUNDED/UNGROUNDED data with my own fictional FAQ of a Bengali medical VQA dataset release ("BanglaMed-Mini"), 5 items: license, size, access, citation, split. All facts invented.
  - Split: train 16 (10 grounded, 6 ungrounded), valid 8, test 24 (15 grounded, 9 ungrounded). Paraphrases split by template, so no question appears in two splits. Test also has 2 grounded questions per item with phrasing never seen in training.
  - JSON format `{"answer": str, "needs_review": bool}` unchanged. System prompt and refusal sentence both English so the scorer and training target agree.
  - Scoring additions: `needs_review` correctness reported separately from the full content check, and ungrounded questions scored per category.
  - Config: Qwen/Qwen3.5-0.8B, LoRA r=16, alpha=32, lr 2e-4, 20 epochs (40 steps), seed 42, greedy decoding, T4. Train time 244 s, final loss 0.014.
  - How the "ungrounded" questions were prepared: hand-written, 18 questions in three categories (6 each, split 2 train / 1 valid / 3 test):
    1. `unrelated`: project trivia with no FAQ topic (PI, funding, GPU).
    2. `adjacent`: about the dataset, but a topic no FAQ item covers (annotators, leaderboard, image resolution).
    3. `near_miss`: shares topic and vocabulary with an FAQ item but asks for a detail it does not give (e.g. "can I redistribute under a different license?").

    I checked that none of the 18 is answerable or derivable from the 5 FAQ sentences (dropped "how many validation samples?" because 10% of 2,400 is derivable). I expected `near_miss` to be hardest, since surface similarity invites a confident answer.
  - Method note: my first run of the course notebook gave identical before/after numbers because the training cell ran before the BEFORE evaluation, so the adapter was already attached (Caution 1 on the slides). In this version BEFORE eval runs first, cells assert no LoRA layers exist before eval and training, and Section 5b regenerates BEFORE outputs with `disable_adapter()`; the check passed.

- Results (comparison tables or numbers):

  Test set, n=24.

  | | format_rate | content_rate | both_rate | grounded needs_review acc | ungrounded needs_review acc |
  |---|---|---|---|---|---|
  | before | 1.00 | 0.25 | 0.25 | 0.53 (8/15) | 0.89 (8/9) |
  | after | 1.00 | 0.83 | 0.83 | 0.80 (12/15) | 1.00 (9/9) |

  Ungrounded refused correctly by category (before -> after, 3 test questions each): unrelated 3/3 -> 3/3, adjacent 3/3 -> 3/3, near_miss 2/3 -> 3/3. Regressions: none.

  - Format was never the problem: the base model already produced valid JSON on 24/24.
  - content_rate overstates the gain. The scorer requires the exact refusal sentence, so the base model scores 0/9 on ungrounded content even though it flagged 8 of them. Judged on `needs_review` alone over all 24: 16/24 (0.67) -> 21/24 (0.88).
  - Remaining failures (4, all grounded): 3 over-refusals ("What is the process to obtain the dataset?", "Can I download it directly without a request?", "Can I see the test set labels?" got the refusal sentence although the FAQ answers them); 1 wrong content with the right flag ("What percentage of the data is used for testing?" -> "Test answers are withheld.", missing 10%).
  - Base model errors were the opposite: `needs_review: true` on 7 of 15 grounded questions, and a confident "2,400" (`needs_review: false`) for the near-miss "How many unique images are there?".
  - Knowledge questions (3 unseen ungrounded + 1 control): before, all 3 flagged `needs_review: true` but the text still asserted a guess ("No" for Hindi translation; "Yes, you can use the dataset for a thesis defense..."). After, all 3 gave the canonical refusal. Control (license) correct both times.
  - Limitations: one run, one seed, 24 questions, so one question = 4 points (33 within a category). near_miss 2/3 -> 3/3 is a hint, not a finding. The 100% ungrounded score comes with 3 grounded over-refusals, so part of it is a lean toward refusing unfamiliar phrasing. Loss 0.014 after 20 epochs on 16 examples means the training set was largely memorized.

- Discussion of when fine-tuning is and is not suitable:

  
  - Suitable when the target is a fixed output behavior over a small, stable question set: the refusal sentence and the license/size/citation answers were learned reliably from 16 examples, and the JSON format cost nothing.
  - Not suitable for learning facts or a grounding check: the model fit the training phrasings and failed on new phrasing (the 3 over-refusals), the mirror image of the base model's errors. Nothing in the adapter checks that the FAQ supports an answer.
  - Not suitable when facts change: they are baked into the weights and need retraining, whereas retrieval plus a support check updates by editing one FAQ line.
