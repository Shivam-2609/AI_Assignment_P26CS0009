AI REASONING SYSTEM
========================================

Main file: reasoning_agent.ipynb
Prediction file: submission.csv

The evaluation input is an internal representation of a knowledge base and
query. The notebook applies classical reasoning and returns a Boolean flag.
All implementation code is included in the notebook; no model training or
generative-AI API is needed for inference.

1. ENVIRONMENT AND SETUP
------------------------
Use Python 3.11 with Jupyter Notebook, JupyterLab, or VS Code's Jupyter support.
The supplied notebook was saved using Python 3.11.15. Select your own Python
kernel; the saved kernel display name "ai" is specific to the author's setup.

For a fresh Jupyter installation, run these commands in a terminal:

    python -m pip install notebook ipykernel datasets huggingface_hub
    python -m notebook

Launch Jupyter from the folder containing reasoning_agent.ipynb, then open it
and select the Python environment in which the packages were installed.
If using an existing notebook environment, install the optional data-loading
packages in that kernel with a temporary cell:

    %pip install datasets huggingface_hub

The reasoning engine and local CSV/JSON/JSONL loading use Python's standard
library. The datasets and huggingface_hub packages are needed for Hugging Face
sources and Parquet files. Hugging Face loading requires access to the selected
repository. A GPU is not required.

2. RUN ONE PRIVATE LOCAL DATASET
--------------------------------
Edit the FIRST configuration cell. Replace the three public datasets with the
private input file in slot 1 and None in all remaining slots. For example:

    DATASET_1 = "private_test.jsonl"
    DATASET_2 = None
    DATASET_3 = None
    DATASET_4 = None
    DATASET_5 = None
    DATASETS = [DATASET_1, DATASET_2, DATASET_3, DATASET_4, DATASET_5]
    OUTPUT_CSV = "submission.csv"
    LABEL_COLUMN = "auto"
    COLUMN_MAP = None
    HF_SPLIT = None
    HF_CONFIG = None
    HF_REVISION = None
    INSTALL_HF_PACKAGES = False
    HF_LOGIN = False
    TIMEOUT_SECONDS = 2.0
    PROGRESS_EVERY = 25

Use an actual input FILE, rather than a folder. Examples:

    DATASET_1 = "private_test.csv"
    DATASET_1 = "private_test.json"
    DATASET_1 = r"C:\TA\datasets\private_test.jsonl"

Choose ONE of these examples. Relative paths are resolved from the notebook's
current working directory. To check that directory, run Path.cwd() after the
imports cell, or use an absolute input path.

After editing, restart the kernel and run ALL cells from top to bottom.
In Jupyter, use "Restart Kernel and Run All Cells" (wording may vary).
The final cell loads the input, produces predictions, prints the available
metrics, and writes submission.csv. Read the printed absolute output path.

Reference answers are optional. An unlabeled private test still produces the
prediction CSV and timing; accuracy is unavailable without reference labels.

3. USE A HUGGING FACE DATASET
----------------------------
Starting from the configuration above, replace DATASET_1 with a repository ID:

    DATASET_1 = "jise/ruletaker"
    HF_SPLIT = "test"

For a private repository, replace that public example with the TA's repository:

    DATASET_1 = "your-organization/your-private-dataset"

Keep DATASET_2 through DATASET_5 as None for a single-source evaluation, and
rerun the first cell and all subsequent cells after making changes.

Hugging Face settings:
  HF_SPLIT = None          Load all available splits in the selected manifest.
  HF_SPLIT = "test"        Load only the named split, if it exists.
  HF_SPLIT = ["train", "validation"]   Load these whole splits.
  HF_CONFIG = None         Use the repository's default configuration.
  HF_REVISION = None       Resolve and pin the current commit for this run.
  HF_LOGIN = True          Prompt for Hugging Face authentication when needed.

Set HF_CONFIG or HF_REVISION explicitly if the evaluation specifies a named
configuration or revision. HF_SPLIT accepts whole split names, not slices such
as "test[:100]". If a repository has no inspectable split manifest, export the
required split as a local file or use HF_SPLIT=None to inspect repository files.
HF split settings do not filter local files.

INSTALL_HF_PACKAGES=True allows the notebook to install missing Hugging Face
packages automatically. Otherwise, install them in the selected kernel first.
HF_LOGIN=False uses existing access credentials where available.

4. INPUT FORMAT AND COLUMN NAMES
--------------------------------
Each input record must contain its own knowledge base and query. For formal
logic, the simplest supported fields are:

  id        Unique string or integer identifier.
  premises  List of symbolic formula strings.
  query     One closed symbolic formula to test for entailment.
  label     Optional reference answer, used only for scoring.

Example private_test.json (expected predictions: q1=True, q2=False):

    [
      {
        "id": "q1",
        "premises": ["Young(Bob)", "forall x. (Young(x) -> Smart(x))"],
        "query": "Smart(Bob)"
      },
      {
        "id": "q2",
        "premises": ["Young(Bob)", "forall x. (Young(x) -> Smart(x))"],
        "query": "Smart(Alice)"
      }
    ]

For JSONL, store one record per line without the surrounding array or commas
between records. For CSV, store the premise list as JSON text inside a quoted
CSV cell. The equivalent private_test.csv is:

    id,premises,query
    q1,"[""Young(Bob)"", ""forall x. (Young(x) -> Smart(x))""]",Smart(Bob)
    q2,"[""Young(Bob)"", ""forall x. (Young(x) -> Smart(x))""]",Smart(Alice)

Use UTF-8 input files. JSON, JSONL, and CSV are recommended; the loader also
supports TSV, NDJSON, and Parquet.

Supported formal syntax includes not, and, or, ->, <->, xor, forall, exists,
equality (=), and inequality (!=). For example:

    forall x. ((Young(x) and Kind(x)) -> Successful(x))
    exists x. Smart(x)
    not Smart(Alice)

Predicate and constant names are case sensitive: Bob and bob are different.
Quantify variables explicitly and parenthesize compound formulas. A period or
colon after the quantified variable gives scope over the following formula;
otherwise the quantifier binds the next unary or parenthesized formula.

Recognized alternatives include premises-FOL/conclusion-FOL (FOLIO), symbolic
facts/rules, supported structured atoms/rules/clauses/ASTs, and explicit Boolean
structural causal models. The public adapters also recognize RuleTaker's
Facts:/Rules: text and CounterBench's given_info/question fields. Input must
match a supported representation; arbitrary natural-language text or arbitrary
JSON structures are not automatically equivalent to formal premises/query.

If private column names differ, set COLUMN_MAP in the first cell. For example,
for test_id, kb_formulas, and query_formula columns:

    COLUMN_MAP = {
        "id": "test_id",
        "premises": "kb_formulas",
        "query": "query_formula"
    }

Mapping changes field names, not the meaning or syntax of their contents.
Keep COLUMN_MAP=None when the standard fields or recognized aliases are used.

LABEL_COLUMN="auto" detects a complete, valid reference column such as label,
flag, answer, or gold. For a custom reference column, use its exact name:

    LABEL_COLUMN = "expected_result"

LABEL_COLUMN=None disables scoring. Accepted reference values include True/
False, Yes/No, 1/0, and Uncertain. Uncertain maps to False for binary entailment.
Labels are not required to produce predictions and do not determine inference.

5. MULTIPLE DATASETS AND GENERATED FILES
---------------------------------------
Up to five input sources can be configured. Fill slots consecutively: processing
stops at the FIRST None or empty slot. For example:

    DATASET_1 = "private_rules.json"
    DATASET_2 = "private_fol.csv"
    DATASET_3 = None
    DATASET_4 = None
    DATASET_5 = None

The common settings, including HF_SPLIT and COLUMN_MAP, apply to each active
slot. For a straightforward evaluation with one private file, use only slot 1.

For one dataset, predictions are written directly to OUTPUT_CSV. With multiple
datasets, OUTPUT_CSV contains the combined predictions and additional dataset
CSVs are written under <output-stem>_datasets/. A dataset with multiple populated
splits/groups also produces a <output-stem>_splits/ folder beside its CSV.

The supplied public-run output layout is:

    submission.csv
    submission_datasets/
      01_jise_ruletaker/
        submission.csv
        submission_splits/       (train, test, ood_test CSVs)
      02_CounterBench_CounterBench/
        submission.csv
        submission_splits/       (alpha and backdoor CSVs)
      03_yale-nlp_FOLIO/
        submission.csv
        submission_splits/       (train and validation CSVs)

The supplied submission_datasets folder contains generated public predictions.
It is not needed to run a new private input. Rerunning writes fresh predictions
to the configured paths. Change OUTPUT_CSV to preserve a separate run.

Every prediction CSV has exactly these columns:

    id,flag
    q1,True
    q2,False

Unique supplied IDs are preserved. Missing IDs are generated automatically.
Repeated IDs across independent partitions or datasets receive prefixes to
avoid collisions; duplicate IDs within the same partition are rejected.

6. METRICS, TIMEOUTS, AND TRACES
-------------------------------
True indicates that the query was established as entailed. For a successfully
resolved query, False indicates non-entailment. Unsupported/malformed input or
an exhausted reasoning budget also produces a False fallback with resolved=False.
Check the printed unresolved count and categories when interpreting accuracy.

TIMEOUT_SECONDS=2.0 is the default per-query reasoning budget. It can be changed
for a specified evaluation budget. Reported inference time sums per-record
prediction durations, including routing/parsing/reasoning; it excludes dataset
loading/downloads and CSV writing. Total notebook runtime can therefore be longer.

After a run, results and summary remain available in the notebook. To inspect a
reasoning trace or unresolved input, run an additional cell:

    show_trace(results[0])
    for row in results:
        if not row["resolved"]:
            print(row["id"], row.get("category"), row["reason"])

The default run prints diagnostics but writes prediction CSVs only; it does not
automatically save separate diagnostic or failure-report files.

7. REPRODUCE THE SUPPLIED PUBLIC RUN
-----------------------------------
Use the submitted defaults in the first cell:

    DATASET_1 = "jise/ruletaker"
    DATASET_2 = "CounterBench/CounterBench"
    DATASET_3 = "yale-nlp/FOLIO"
    DATASET_4 = None
    DATASET_5 = None
    HF_SPLIT = None
    HF_CONFIG = None
    HF_REVISION = None
    LABEL_COLUMN = "auto"
    COLUMN_MAP = None
    TIMEOUT_SECONDS = 2.0

Restart the kernel and run all cells. Hugging Face sources require the packages
and repository access described above. The loader prints the actual revisions
and source files used. The supplied output recorded these revisions:

    jise/ruletaker:            b6e36448e186cde51052e978da0465d84da06f31
    CounterBench/CounterBench: c6225dfa00b7a89ccfeae67797b9a6d80e48e44c
    yale-nlp/FOLIO:            4c1f14bf39ca6a6de242bd8f75ef313b4b436818

HF_REVISION=None does not automatically restore these recorded snapshots;
later repository updates can change the input. To pin separate repositories,
each dataset slot can use its own options, for example:

    DATASET_1 = {
        "dataset": "jise/ruletaker",
        "hf_revision": "b6e36448e186cde51052e978da0465d84da06f31"
    }

Use the corresponding repository's SHA for each slot, rather than one global
SHA for all three repositories.

Latest results (including training and other available public splits):

    Dataset          Questions   Correct   Accuracy   Inference seconds
    RuleTaker             1625      1600     98.46%            0.569105
    CounterBench          1200      1135     94.58%            0.350887
    FOLIO                 1204       947     78.65%           25.610071
    Combined              4029      3682     91.39%           26.530062

These are combined public-data results, not private-test or held-out-only
accuracy. The supplied run has 247 unresolved cases. CounterBench's query-free
meta_model support rows are inspected and excluded; its 1200 question rows are
included. FOLIO's 393 Uncertain labels map to False for binary scoring.
Timing and timeout-dependent predictions may vary with the evaluation machine.

8. QUICK TROUBLESHOOTING
------------------------
File not found: check Path.cwd(), use an absolute path, and specify an input
file rather than the submission_datasets output folder.

Missing datasets/huggingface_hub: install them in the active notebook kernel
or set INSTALL_HF_PACKAGES=True before rerunning.

Only one dataset ran: check for a None/empty slot before the remaining inputs.
Use the Python value None, without quotation marks, for unused slots.

Accuracy unavailable with LABEL_COLUMN="auto": reference labels are absent,
incomplete, invalid, or conflicting. Predictions can still be used. For a
custom label name, set LABEL_COLUMN explicitly and ensure every record has a
valid value. A missing/invalid explicitly named reference raises an error after
the individual prediction CSV is saved; final aggregation may not complete.

Input/parse errors: check COLUMN_MAP, formula syntax, quantifier scope, and the
unresolved reasons in results. A True/False CSV alone does not show whether a
prediction was resolved or was a fallback.

If a later dataset fails, earlier per-dataset CSVs can remain, but the final
combined run has not completed. Check for FINAL COMBINED RESULTS and the
printed output path after a successful run.
