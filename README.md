# COSMIC: Leader-Driven Context-Oriented Collaboration Between Agents

Research code and recorded outputs for the COSMIC multi-agent coordination model, as
published in *IEEE Transactions on Computational Social Systems* (the paper is included:
`COSMIC_Leader-Driven_Context-Oriented_Collaboration_Between_Agents.pdf`).

COSMIC runs a small crew of specialist LLM agents around one Leader. Every agent keeps a
bounded memory of the conversation, a speaker-selection function decides who speaks next,
the Leader checks in on a schedule, and the run ends when the Leader judges the work done.

## Layout

| Path | What |
|---|---|
| `notebooks/tasks/` | The canonical runs, one notebook per task: healthcare, mathematics, bandwidth allocation, resource allocation, robot warehouse, translation, story writing, scaling, researcher, business. All follow the same template (below). **`task-healthcare.ipynb` is the reference implementation.** |
| `notebooks/autogen modified/` vs `notebooks/autogen default/` | The same tasks in the modified AutoGen setup and in stock AutoGen, side by side, for comparison. |
| `notebooks/misc/` | A working copy. |
| `Task Outputs/` | Recorded outputs of the notebook runs, grouped by task (transcripts, screenshots, comparison runs against CrewAI sequential/hierarchical setups). |
| `python/experiments/` | Ad-hoc single-file experiments (AutoGen, CrewAI, LangChain variants; sequential-trip and group-chat prototypes). Not a package. |
| `python/legacy/` | Earlier top-level files, kept as a baseline. |
| `tools/search_tools.py`, `Code/tools/search_tools.py` | Serper-based web search wrappers used by the notebooks and experiments (duplicates). |
| `scripts/` | Helpers: `patch_notebooks_add_repo_root.py` (adds the `sys.path` bootstrap cell so notebooks can import `tools/`), `make_autogen_default.py` (derives the stock-AutoGen variants), `termination_sensitivity_sweep.py` (parameter sweep over the check-in interval and the consistency/stability thresholds), `redact_secrets.py`, `print_versions.py`. |
| `requirements.txt` | A broad superset of the Python dependencies, pinned to mid-2024 versions (AutoGen, CrewAI, LangChain, Chroma...). |

## The notebook template

Every task notebook has the same shape; only the agents, prompts and scripted data change.

- **`MemoryAgent(ConversableAgent)`** overrides `receive` and `send` to append every
  non-tool message to its own `self.memory` list, and builds its context as
  `[system_message] + self.memory[-10:] + messages`: each agent has its own last-10 window,
  separate from AutoGen's shared `GroupChat.messages`.
- **A fixed crew**: one `Leader`, the task specialists, a `Result_Provider` (scripted to
  inject pre-written test results mid-run), a `Tool_executor` (a plain `ConversableAgent`
  that only executes tools) and a `user_proxy` that terminates on `TERMINATE`.
- **`allowed_speaker_transitions_dict`**: an explicit adjacency map passed to `GroupChat`
  with `speaker_transitions_type="allowed"`. Every agent may speak to every other agent
  except `Tool_executor`; only `user_proxy` hands off to `Tool_executor`.
- **`custom_speaker_selection_func(last_speaker, groupchat)`**: turn 0 is always the Leader;
  a fixed call count triggers the `Result_Provider`; every 7th call is a forced Leader
  check-in; otherwise a small model call picks the next speaker from a hard-coded candidate
  list ("ONLY RESPOND WITH THE NAME OF THE AGENT").
- **Termination**: `is_termination_msg` fires on `TERMINATE`; the Leader's system prompt
  ends with the instruction to say it when the task is done.
- **Post-run**: `save_conversation_to_file` dumps `groupchat.messages`, and
  `structure_logs_with_local_llm` feeds the dump to a separate local model for a structured
  summary.
- **Models**: the notebooks target **Ollama** (e.g. `qwen2.5:72b`) through an
  OpenAI-compatible endpoint configured in `llm_config`; they are not written for the
  OpenAI API.

## Running the notebooks

1. Python 3.10, then `python -m venv .venv`, activate it, `pip install -r requirements.txt`.
2. Copy `.env.example` to `.env` and set `SERPER_API_KEY` (for web search) and your model
   endpoint. In the notebooks, replace the placeholders `YOUR_SERPER_API_KEY` and
   `http://YOUR-OLLAMA-HOST:11434` with your own values (or read them from the
   environment); the originals pointed at private endpoints that no longer exist.
3. Run `python scripts/patch_notebooks_add_repo_root.py` once so the notebooks can import
   from `tools/`, then open a notebook in `notebooks/tasks/` and run it top to bottom.

Keep the template when adding a task: per-agent `MemoryAgent`, scheduled agents at fixed
call counts, an explicit allowed-transitions map, and a Leader that terminates.
