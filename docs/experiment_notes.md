# Experiment notes

This document separates the original report's observations from the configuration and outputs retained in the supplied notebook. It records limitations and the publication cleanup so that the portfolio does not imply a stronger experiment than the source material supports.

## Evidence available

| Source | What it supports |
| --- | --- |
| `Assignment3.docx` | Dataset rationale, adaptations to the starter workflow, 10 math and 10 code prompt/response pairs, reflections, and a classical-baseline summary |
| `MMA_606_Assignment_4.ipynb` | Final Code-Feedback training configuration, saved training diagnostics, one retained mutable-default prompt and two responses, classifier code, and its saved confusion-matrix image |
| Cleaned notebook in this repository | The same core experiment code with documented publication and inference-state hygiene changes; outputs cleared |
| `assets/baseline_confusion_matrix.png` | The original notebook's embedded figure, extracted without redrawing or rerunning it |

The report uses `[1]` for a prompt, `[2]` for a base response, and `[3]` for an adjusted response. Its second experiment heading repeats the ORCA dataset name; the programming prompts and reflection identify that section as the Code-Feedback experiment. The original report's private document-sharing link is not reproduced here.

## Research question and design

The project asks whether a small supervised adaptation can change response completion, attention to mathematical wording, and the thoroughness of code repair, and whether those changes appear in exploratory transfer examples. The author selected mathematics and programming based on prior experience teaching secondary mathematics, VB programming, and GMAT logic.

The report describes two dataset-specific adapters:

- **Math:** `HuggingFaceH4/orca-math-word-problems-200k`, using question/answer conversations. The report describes removing the redundant `question` and `answer` columns for training.
- **Code:** `HuggingFaceH4/Code-Feedback`, using multi-turn programming conversations. The notebook loads `train_sft` and removes `id`, retaining `messages`.

These are separate described experiments, not sequential training of one adapter on both datasets. Only the final code configuration survives in full. Commented alternatives and report prose are insufficient to reconstruct an exact historical math run, so this edition does not invent a second executable configuration.

## Saved Code-Feedback configuration

| Setting | Value in source notebook |
| --- | --- |
| Checkpoint | `unsloth/qwen3-14b-unsloth-bnb-4bit` |
| Output directory | `qwen3-14b-unsloth-bnb-4bit-SFT2` |
| Loaded dataset rows | 65,383 in saved dataset display |
| Attention implementation | `sdpa` |
| Model and quantization compute dtype | `torch.bfloat16` |
| Requested quantization | `load_in_4bit=True`, `nf4`, double quantization enabled |
| LoRA rank / alpha | 32 / 32 |
| Target modules | `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj` |
| Per-device batch / accumulation | 1 / 4; nominal effective batch of 4 on one GPU |
| Training steps / warmup | 88 / 5 |
| Learning rate / optimizer | `2e-4` / `paged_adamw_8bit` |
| Maximum sequence length | 2,048 |
| Liger kernel / activation offloading | Enabled / disabled |
| Evaluation dataset | None supplied to `SFTTrainer` |
| Generation budget | 512 new tokens for each model |

The source loads a prequantized checkpoint. Its saved warning states that the checkpoint's own quantization configuration takes precedence over the supplied configuration. The table records the code's requested settings; it is not an independent inspection of checkpoint internals. Trainer defaults and model/dataset revisions were not frozen. In particular, the source sets bfloat16 for loading and quantization computation, but does not explicitly set `bf16=True` in `SFTConfig`.

The training step limit is not a complete pass over the loaded split. Do not read 65,383 loaded rows as 65,383 examples trained on, or infer the original math subset size from the code run.

### Diagnostics retained in the original output

- GPU: NVIDIA A100-SXM4-40GB; reported total memory 39.494 GiB.
- Training completed 88 of 88 optimizer steps.
- Training runtime: 376.2932 seconds (6.27 minutes).
- Peak reserved CUDA memory: 23.549 GiB; pre-training reading: 21.805 GiB; reported difference: 1.744 GiB.
- Last displayed training loss: 0.657245 at step 88. This is a training-step diagnostic, not a held-out performance score.

The original code divides bytes by powers of 1,024 but labels the output “GB”; the values above use GiB. CUDA peak-reserved counters and earlier allocations are stateful, so the difference is not an isolated measurement of adapter memory. The report's “1,000 samples in under two minutes” statement does not match the saved code-run diagnostic and is not used as a performance claim here.

## Qualitative probe record

The following summaries describe the responses in the report. They are not newly generated answers, automated scores, or evidence that the test questions were absent from training. The supplied materials do not include an execution-based test suite or recorded test results for generated programs.

### Mathematics adapter

| ID | Probe | Base response recorded | Adapted response recorded |
| --- | --- | --- | --- |
| M1 | Manuscript typing and revisions | Repetitive calculation; ends before total | Completes with $680 |
| M2 | Profit is 40% of revenue; express as percent of cost | 66.67% | 66.67% |
| M3 | 20% discount with 20% profit on cost | Confuses cost and list price; ends unfinished | Correct relation and 50% |
| M4 | Round trip at 40 and 60 km/h | 48 km/h | 48 km/h |
| M5 | Selling price of 15 items equals cost of 20 | 33⅓% | 33⅓% |
| M6 | Doubling selling price triples profit | Incorrect 33.33% | Correct 100% |
| M7 | Profit is 20% of selling price; express as percent of cost | Incorrect 20% | Correct 25% |
| M8 | 320% initial markup; cost rises 25% | Approximately 70.24% of selling price | Approximately 70.24% |
| M9 | Repeated milk/water replacement | 9:1 | 9:1 |
| M10 | Bat-and-ball cognitive-reflection example | $0.05 | $0.05 |

### Code adapter

| ID | Probe | Base response recorded | Adapted response recorded |
| --- | --- | --- | --- |
| C1 | FizzBuzz | Repetitive explanation; truncated before complete implementation | Complete implementation |
| C2 | Two Sum | Dictionary-based solution | Same substantive approach |
| C3 | Binary-search repair | Upper-bound change leaves an inconsistent algorithm | Consistent inclusive bounds, loop condition, and pointer updates |
| C4 | Factorial infinite-loop repair | Adds `n -= 1` | Adds `n -= 1` |
| C5 | Explain/comment a palindrome function | Comments and explanation | Comments and explanation |
| C6 | Optimize duplicate detection | Iterative set solution | Set-length comparison; both average O(n) |
| C7 | Deadlock explanation and prevention | Longer explanation and four named strategies | Shorter explanation and overview |
| C8 | Java in-place character-array reversal | Two pointers plus demonstration | Two-pointer method |
| C9 | VB.NET factorial with negative-input handling | Program handles requested negative-input case | Similar program handles that case |
| C10 | Python mutable default argument | Repeats input code without explaining outputs | Also repeats input code without explaining outputs |

C3 is described as a coherent algorithm repair, rather than a count of four independent bugs: half-open and inclusive-bound binary search use different valid conventions. C9 is an exploratory cross-language example, not proof of unseen-language generalization or production-quality input handling. C10 establishes a failure to answer that particular prompt; it cannot identify an internal conceptual deficit.

### What can be concluded

Some selected responses become more concise or complete within the stated output budget. Several profit-language errors disappear in the reported math comparisons, and the code comparison shows a more complete binary-search repair. Other pairs are materially similar, and the mutable-default question remains unanswered.

The project does not measure statistical significance, general coding accuracy, contamination-free transfer, or internal reasoning. Reduced visible `<think>` text is an output observation. The report's suggested mechanisms, such as learned linguistic sensitivity or a reward for deeper checking, are interpretations; this notebook implements supervised fine-tuning, not a reward-learning experiment.

## Classical TF-IDF baseline

The baseline labels original question–answer pairs as 1 and randomly mismatched pairs as 0. A mismatch is a synthetic negative label, not a mathematical proof that an answer is incorrect. Features concatenate the question, ` [SEP] `, and answer, use `TfidfVectorizer(max_features=5000, stop_words='english')`, and feed `LogisticRegression(max_iter=1000)`. A most-frequent dummy classifier is also evaluated with five-fold cross-validation on the training features; its scores are assigned but not displayed in the saved notebook.

The report records **41.8% test accuracy** for logistic regression and a 75/25 split. The original figure is retained in [the README](../README.md#classical-baseline). Its confusion matrix has true classes as rows and predicted classes as columns, ordered mismatched then original:

```text
[[206, 294],
 [288, 212]]
```

The figure contains 1,000 test pairs, with 500 in each true class: `(206 + 212) / 1000 = 41.8%`. A constant majority-class prediction would achieve 50% on those labels. This classifier result should not be compared numerically with the language model's qualitative generation results.

### Source discrepancy and limitations

- **Sample counts differ:** the report says 1,000 positive plus 1,000 negative pairs (2,000 total). The saved code requests `train_sft[:2000]` and creates one positive and one negative pair per row, implying 4,000 total pairs and a 1,000-pair test set if run as written. The figure's 1,000 test pairs agree with the current code's implied test size. The source is an interactive notebook with edited cells; agreement in size does not prove that every current setting produced the displayed artifact.
- **Feature leakage:** TF-IDF is fitted on all texts before train/test splitting, so the test corpus influences vocabulary and inverse-document-frequency weights.
- **Related examples can cross the split:** pairs derived from the same questions or answers are not grouped before splitting.
- **Partial seeding:** `random.seed(10086)` controls mismatched-answer selection, but `train_test_split(..., test_size=0.25, stratify=labels)` has no `random_state`.
- **No repeated evaluation:** a single reported accuracy does not quantify uncertainty. The balanced majority-class reference is 50%, but no repeated-seed or significance analysis was performed.

These steps are preserved in the cleaned notebook to retain the experiment's substance. A future study should split by source problem before generating pairs, fit TF-IDF only on training text using a pipeline, fix and report seeds, and rerun evaluation. That would be a new result, not a correction to the historical 41.8%.

## Reproduction boundaries

The original installation was unpinned, the notebook has no full environment freeze, and dataset/model revisions were not recorded. The final artifact therefore cannot guarantee exact numerical reproduction. `requirements.txt` lists the dependencies needed by this edition; it does not assert a tested version combination. Training requires a suitable Linux/CUDA environment, including a compatible PyTorch, bitsandbytes, and Liger stack.

The original execution counts are non-monotonic. In particular, adapter attachment was executed before the retained “base” response was regenerated. PEFT may mutate the supplied base model when attaching an adapter, so the saved base/tuned labels are not independently verified by execution order. The cleaned notebook explicitly disables the adapter for base generation, using [PEFT's documented context manager](https://huggingface.co/docs/peft/package_reference/peft_model#peft.PeftModel.disable_adapter); this improves future comparisons but does not retroactively validate earlier results.

The generation code preserves the original prompt and 512-token cap. Sampling settings, random generation seeds, and thinking-mode controls were not explicitly fixed in the source. No GPU training or model inference was rerun for this portfolio edition. Structure, syntax, preservation of core configuration, and sensitive-content checks were performed before publication.

## Source record and publication cleanup

This project originated from an **MMA 606 course assignment**. Its training notebook was adapted from the **course-provided starter workflow**, with the upstream [Hugging Face TRL SFT LoRA/QLoRA notebook](https://github.com/huggingface/trl/blob/main/examples/notebooks/sft_trl_lora_qlora.ipynb) credited in the original opening cell. The experiment should be attributed as adaptation and analysis, not as a training framework authored from scratch.

Source fingerprints (SHA-256):

```text
MMA_606_Assignment_4.ipynb
02475fe215d90a53ebf06e6026bc47fb994767d7e17908fb3d1441f8e59bfedc

Assignment3.docx
0d7e5638ca2f53bdec3741ec8cfff1f9f24b14c5185b39610e251c6b0bf9dace
```

Publication changes:

1. Replaced assignment directions, model menus, stale account references, and generic template text with project-specific context and attribution.
2. Cleared cell outputs, execution counters, login widgets, and runtime metadata. No hardcoded credentials were identified during source review; the published notebook contains no login widget state or embedded token.
3. Disabled automatic Hub publishing and remote experiment tracking; adapters are saved and reloaded locally.
4. Removed the optional merge/vLLM serving appendix and abandoned commented alternatives. They do not support the reported experiments and contained stale publication destinations. Null execution counters alongside retained outputs in the source appendix do not establish a reliable execution history.
5. Made base inference explicitly run without the adapter when comparing responses, addressing shared-model state on repeated cell execution.
6. Corrected the effective-batch-size comment and retained the model, training data selection, LoRA/training parameters, generation budget, final prompt, and baseline method.
7. Extracted the original confusion-matrix figure as a separate asset and documented the saved code/report discrepancies without fabricating a replacement result.

The original notebook and Word report are kept unchanged outside this public repository. No model weights, full training datasets, authentication files, personal document-sharing links, or raw runtime logs are included.
