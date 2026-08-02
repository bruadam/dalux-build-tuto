# Dalux Build Tutorial

A 5-hour, hands-on Jupyter notebook tutorial for the
[`dalux_build`](https://pypi.org/project/dalux-build/) Python client (Dalux
Build's REST API — files/BIM, tasks, forms, inspections, work packages, …).

## Setup

```bash
uv sync                        # installs dalux-build, pandas, python-dotenv, Jupyter, and nbstripout
uv run nbstripout --install --attributes .gitattributes   # strip notebook outputs before every commit
cp .env.example .env            # then fill in your real DALUX_API_KEY / DALUX_BASE_URL
uv run jupyter lab tutorials/   # open the notebooks
```

Your API key lives in the Dalux Build UI under **Settings → Integrations → API
Identities**. `.env` is git-ignored.

### Notebook outputs never get committed

`nbstripout` is registered as a git filter (see `.gitattributes`) that strips
cell outputs and execution counts from `*.ipynb` files at `git add`/`commit`
time — the version on disk still shows your run's output, but only clean source
ever reaches git. Run the `nbstripout --install` command above once after
cloning (it's local to your `.git/config`, so every fresh clone needs to re-run
it).

## Tutorials (`tutorials/`)

One notebook per hour, meant to be run in order:

| Notebook                                                                                           | Covers                                                                                                                                                      |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`hour_1_setup_and_projects.ipynb`](tutorials/hour_1_setup_and_projects.ipynb)                     | Credentials, `create_client()`, the `DaluxClient` namespaces, projects/companies/users, error handling                                                      |
| [`hour_2_files_and_folders.ipynb`](tutorials/hour_2_files_and_folders.ipynb)                       | File areas → folders → files, building a folder tree, downloading (single/bulk/filtered), chunked uploads                                                   |
| [`hour_3_tasks_forms_and_workpackages.ipynb`](tutorials/hour_3_tasks_forms_and_workpackages.ipynb) | Tasks, task change history, forms, work packages, the other read-only resources, reusable pagination/search utilities, and a capstone project status report |
| [`hour_4_bonus_graphs.ipynb`](tutorials/hour_4_bonus_graphs.ipynb)                                 | Bonus: charting `to_dataframe=True` results with `matplotlib` — companies, work packages, the Hour 2 folder tree, task status — combined into one dashboard |
| [`hour_5_webhook_server.ipynb`](tutorials/hour_5_webhook_server.ipynb)                             | Resource-scoped dashboards and the embedded `dalux.webhook_server` — task timelines, scheduled change jobs, and freshness jobs                              |

## Claude Code skill

[`.claude/skills/dalux-build/SKILL.md`](.claude/skills/dalux-build/SKILL.md)
teaches Claude Code how to use every `dalux_build` API class and method
correctly (conventions, pagination, path resolution, file upload/download,
exceptions) — it loads automatically whenever you ask Claude to write or debug
code against this package.
