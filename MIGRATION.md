# Migration: gpt-4o-mini → gpt-6-luna

## What changed

The app used the OpenAI model **`gpt-4o-mini`** to answer questions about a PDF.
It now uses **`gpt-6-luna`**.

Nothing else changed. The app still works the same way, still uses the same
LangChain code, and still uses the same embedding model.

## Why

OpenAI is retiring `gpt-4o-mini` (and also `gpt-4o`, `gpt-4.1` and `gpt-4.1-mini`)
from the API on **January 14, 2027**. After that date, any request that asks for
`gpt-4o-mini` will fail, so this app would stop answering questions.

OpenAI's official deprecations page lists `gpt-6-luna` as the recommended
replacement:

- Page: <https://developers.openai.com/api/docs/deprecations>
  (the old link <https://platform.openai.com/docs/deprecations> redirects there)
- Row in the "Upcoming deprecations" table:

  | Shutdown date | Affected model | Status | Recommended replacement | Notice date |
  |---|---|---|---|---|
  | Jan 14, 2027 | GPT-4o mini (`gpt-4o-mini-2024-07-18`) | Deprecated | `gpt-6-luna` | Oct 14, 2026 |

Model page: <https://developers.openai.com/api/docs/models/gpt-6-luna>

## Every file and line changed

| File | Line(s) | Before | After |
|---|---|---|---|
| `app.py` | 49 | `ChatOpenAI(model_name="gpt-4o-mini", ...)` | `ChatOpenAI(model_name="gpt-6-luna", ...)`, already changed upstream in commit `139efd3` (with an inline comment), so this PR doesn't touch `app.py` |
| `README.md` | 5 | "OpenAI's GPT-4 model" | "OpenAI's GPT-6 Luna model (`gpt-6-luna`)" |
| `README.md` | 37 | "Use the GPT-4 model" | "Uses the GPT-6 Luna model" |
| `README.md` | 60 | "processes them with GPT-4" | "processes them with GPT-6 Luna" |
| `README.md` | 115 | "The model used for answering questions is GPT-4" | "... is GPT-6 Luna (`gpt-6-luna`)" |
| `README.md` | 119 | `model_name="gpt-4o-mini"` (code example) | `model_name="gpt-6-luna"` |
| `MIGRATION.md` | all | *(new file)* | This document |

The README said "GPT-4" even though the code used `gpt-4o-mini`. It now
names the model the app actually uses.

## What did NOT change, and why

- **Python packages (`pyproject.toml`, `uv.lock`)**: no upgrade needed.
  `gpt-6-luna` is a *reasoning* model. Some reasoning models reject settings
  like `temperature`. We checked what the locked version
  (`langchain-openai 0.3.7`, `openai 1.64.0`) sends for `gpt-6-luna`: only
  `model` and `stream`, with no `temperature`. So the old packages work as they are.
  - We did **not** upgrade to LangChain 1.x. That version moved
    `langchain.chains` (used by `create_retrieval_chain` and
    `create_stuff_documents_chain`), so upgrading would mean rewriting code
    that has nothing to do with this migration.
- **`ChatOpenAI` class**: same class, same arguments. Only the model name changed.
- **Embeddings (`text-embedding-ada-002`)**: not on the deprecation list, so
  it was left alone. Changing the embedding model would also change how
  documents are searched, so it should be a separate change.

## Things to know about the new model

- `gpt-6-luna` is a reasoning model. It "thinks" before it answers, using
  medium effort by default, so answers may be a bit slower than before.
- Reasoning tokens are billed as output tokens. Price at the time of writing:
  $0.10 per 1M input tokens and $0.50 per 1M output tokens.
- If you want faster or cheaper answers, you can lower the effort:
  `ChatOpenAI(model_name="gpt-6-luna", reasoning_effort="low", ...)`.
  This migration does not do that, to keep the change small.

## How to test the app

1. **Install the packages** (from inside the `Basic-Rag` folder):

   ```bash
   pip install uv
   ```

   ```bash
   uv sync
   ```

2. **Add your OpenAI API key.** Create the file `scripts/secret.py` (it is
   ignored by git, so it will not be uploaded) containing:

   ```python
   OPENAI_KEY = "sk-...your key..."
   ```

3. **Start the app:**

   ```bash
   uv run streamlit run app.py
   ```

   Your browser opens at `http://localhost:8501`.

4. **Try it:**
   - Upload a PDF. The repo includes `temp.pdf` you can use.
   - Wait for "Processing document..." to appear.
   - Ask a question whose answer is in the PDF. You should get a sensible answer.
   - Ask something that is *not* in the PDF. The model should say it doesn't know.

5. **Confirm the new model was used:** open
   <https://platform.openai.com/usage> and check that the requests show
   `gpt-6-luna`, not `gpt-4o-mini`.

6. **Check that the old model is gone from the code:**

   ```bash
   git grep -n "gpt-4o-mini"
   ```

   The only matches should be the inline comment on `app.py` line 49 and this file.

### What was already tested (without a real API key)

The app's LangChain chain was run against a fake local OpenAI server, using the
same package versions as `uv.lock`. The request sent `model: "gpt-6-luna"`
(with no `temperature`), and the answer came back and was read correctly.
This proves the code and packages work together. It does **not** prove the
quality of real answers, so step 4 above still needs to be done with a real key.
