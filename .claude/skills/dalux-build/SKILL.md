---
name: dalux-build
description: Use when the user is learning or working with the dalux_build package (the Dalux Build API client) in this repo — creating a client, calling any ProjectsApi/FilesApi/TasksApi/etc. method, handling pagination or file paths, uploading/downloading files, or catching Dalux exceptions. This is a tutoring skill — explain and point, don't write the code for them.
---

# dalux_build Python client — tutor mode

`dalux_build` is a typed Python wrapper around the Dalux Build REST API (construction project management:
files/BIM, tasks, forms, inspections, work packages, …). This repo depends on it (see `pyproject.toml`) and has
three teaching notebooks under `tutorials/` (`hour_1_setup_and_projects.ipynb`,
`hour_2_files_and_folders.ipynb`, `hour_3_tasks_forms_and_workpackages.ipynb`).

**This skill exists to help the user learn the client, not to get code written for them.** When it's active:

- **Explain, don't edit.** Describe the concept, name the exact method/class/model involved, and say what needs
  to change and why. Show the relevant snippet as an example to read, not as a diff to apply.
- **Do not use Edit/Write/NotebookEdit to modify the user's notebook or code** as part of this skill's guidance,
  even if you're confident about the fix. Tell them what to change and let them type it in — that's the whole
  point of a tutorial repo. The one exception: the user explicitly asks you to make the change yourself
  ("fix it for me", "just edit the cell") — then it's a normal coding request, not a tutoring one.
- **Point at the source, don't paraphrase from memory.** Send them to the exact file/line in the installed
  package (see "Golden rule" below) so they build the habit of reading the client themselves.
- **Ask before answering** when a question is ambiguous about *which* concept they're stuck on (e.g. pagination
  vs. path resolution vs. auth) rather than guessing and dumping everything.

## Golden rule: verify against the installed source, don't guess

This package evolves. Before relying on a method signature, a model field, or a body shape, check the version
actually installed in the project's venv — do not trust training-data memory of "some Dalux API":

```bash
python -c "import dalux_build, os; print(os.path.dirname(dalux_build.__file__))"
```

Then read the relevant file directly, e.g.:
- `dalux_build/api/<name>.py` — method signatures, docstrings with the exact HTTP path
- `dalux_build/models/<name>/models.py` and `responses.py` — field names, aliases, required vs optional
- `dalux_build/utils/*.py` — pagination, search, path resolution, validation helpers

If a method or field mentioned below doesn't exist when you grep the installed package, the installed version has
drifted from this doc — trust the source, not this file.

## Setup

Credentials load from env vars `DALUX_API_KEY` and `DALUX_BASE_URL` (see `.env.example` in repo root; copy to
`.env`, which is git-ignored).

```python
from dotenv import load_dotenv
load_dotenv()

from dalux_build import create_client
dalux = create_client()  # or create_client(base_url=..., api_key=...) to bypass env vars
```

`create_client()` returns a `DaluxClient` dataclass — one attribute per resource group, all sharing one
authenticated `requests.Session` (`ApiClient`, injects `X-API-KEY` on every call):

| Attribute | Class | Covers |
|---|---|---|
| `projects` | `ProjectsApi` | List/get/create/update projects, project metadata |
| `companies` | `CompaniesApi` | Companies on a project |
| `company_catalog` | `CompanyCatalogApi` | Company-profile-wide company catalog |
| `users` | `UsersApi` | Company & project users |
| `file_areas` | `FileAreasApi` | File areas on a project |
| `folders` | `FoldersApi` | Folders within a file area |
| `files` | `FilesApi` | Files within a file area — browse, search, download |
| `file_upload` | `FileUploadApi` | Chunked upload (create → part → finalize) |
| `file_revisions` | `FileRevisionsApi` | Download historical revision content |
| `tasks` | `TasksApi` | Tasks / issues / approvals / safety observations |
| `forms` | `FormsApi` | Forms and form attachments |
| `work_packages` | `WorkPackagesApi` | Work packages |
| `inspection_plans` | `InspectionPlansApi` | Inspection plans, items, zones, registrations |
| `test_plans` | `TestPlansApi` | Test plans, items, zones, registrations |
| `version_sets` | `VersionSetsApi` | Version sets & their files |
| `project_templates` | `ProjectTemplatesApi` | Project templates on the company profile |

Almost every method takes `project_id` as its first argument (this API is project-scoped).

## Core conventions (apply across every API class)

**Typed responses.** List endpoints return a `*ListResponse` Pydantic model with `.items: List[Model]`,
`.metadata` (`total_items` / `total_remaining_items`), and `.links`. Single-item endpoints return a `*Response`
with `.data: Model`. Fields use Python snake_case with a Pydantic alias for the API's camelCase
(`project.project_name`, alias `"projectName"`). Use `.model_dump()` for a plain dict, `.model_dump(by_alias=True)`
to get the original API field names (e.g. for building a `pandas.DataFrame`).

**"One page" vs "all pages".** For any resource with a lot of items, look for two sibling methods:
- `list_x(...)` / `get_project_x(...)` — a single page as the API returned it
- `get_all_x(...)` — follows the API's bookmark pagination automatically, returns a flat `List[Model]`,
  accepts `verbose=True` to print page-by-page progress

`get_all_folders`, `get_all_files`, `get_all_project_tasks`, `get_all_project_task_changes` all follow this
pattern. Under the hood they call `dalux_build.utils.pagination.paginate(endpoint, client, params, verbose)`,
which is also usable directly against any raw endpoint (including ones this package doesn't wrap yet) — it needs
the low-level `ApiClient`, reachable off any API instance as `dalux.tasks._client` (or construct one directly, see
Advanced usage below).

**Path-based lookups for files/folders.** Anywhere you see `file_area_id_or_path`, you can pass either a real
`file_area_id` **or** a full path string starting with the file area's display name, e.g.
`"Files/4_Design/C07_Geometry/C07.05_BIM"`. Resolution happens via
`dalux_build.utils.path_resolver.resolve_folder_id_from_named_path`, which is cached internally per-call and
supports `verbose=True` to print each resolution step. `folders.get_file_area_tree_by_path` additionally supports
`*` wildcards in path segments.

**Search helpers, not a query language.** `dalux_build.utils.search` provides `find_by_field(items, field, value)`
and `find_all_by_field(...)`, which work transparently across Pydantic models *and* raw `{"data": {...}}` dicts.
Several convenience methods are built on top: `projects.get_project_by_name`, `file_areas.get_file_area_by_name`,
`folders.get_folder_by_name`, `company_catalog.get_company_by_name`.

**Exceptions**, all importable from the top level (`from dalux_build import DaluxError, NotFoundError, ...`):

| Exception | When |
|---|---|
| `DaluxError` | base class — catch this for "anything Dalux-related failed" |
| `NotFoundError` | HTTP 404 |
| `AuthenticationError` | HTTP 401 (bad/missing API key) |
| `RateLimitError` | HTTP 429 |
| `ApiError` | any other 4xx/5xx |
| `ValidationError` | raised **client-side**, before any request, e.g. empty `project_id`/`file_area_id` |

## Files: the trickiest resource group

- Browsing (`list_files`, `get_all_files`) hits `/6.1/.../files`; fetching a single known file ID
  (`get_file(project_id, file_area_id, file_id)`) hits `/5.0/.../files/{fileId}`. `get_file` **overloads** on
  whether `file_id` is given: `get_file(project_id, file_area_id, file_id)` vs
  `get_file(project_id, "Files/folder/.../name.ext")` (path form, no `file_id`).
- Downloads always go through a `File.download_link` using the same `X-API-KEY`, not a plain unauthenticated URL.
- `bulk_download_folder(..., filters=FileNameFilter(...))` supports include/exclude filtering by
  contains/startswith/endswith/extension (see `dalux_build.models.FileNameFilter`), and skips re-downloading a
  file when an identical revision already exists locally.
- Uploading is **compose-it-yourself**, three calls on `dalux.file_upload`:
  ```python
  upload = dalux.file_upload.create_upload(project_id, file_area_id, {"fileName": "x.pdf", "mimeType": "application/pdf"})
  dalux.file_upload.upload_file_part(project_id, file_area_id, upload["uploadGuid"], chunk_bytes)  # loop for big files
  result = dalux.file_upload.finish_upload(project_id, file_area_id, upload["uploadGuid"], {"folderId": target_folder_id})
  new_file_id = result["fileId"]
  ```
  This mutates real project data — flag it clearly to the user and don't run it against a real project without
  their explicit confirmation.

## Tasks are intentionally loose

`Task` (and related change/attachment models) use `extra="allow"` — only `task_id` is guaranteed. This is because
task *type* (issue, approval, safety observation, good practice, …) determines the rest of the schema. Use
`.model_dump(by_alias=True)` to see whatever fields actually came back rather than assuming a fixed shape.
`get_project_tasks(project_id, params={"typeId": "..."})` is a shorthand the client expands into
`$filter=data/type/typeId eq '<typeId>'` automatically; pass `$filter` yourself for anything more complex (it's
OData).

## Advanced: bypassing `create_client()`

For dependency injection / testing, construct the layers directly instead of the convenience factory:

```python
from dalux_build.configuration import Configuration
from dalux_build.api_client import ApiClient
from dalux_build.api import ProjectsApi, TasksApi

config = Configuration(base_url="https://<company>.dalux.com/api", api_key="...")
api_client = ApiClient(config)
projects = ProjectsApi(api_client)
```

## How to respond when the user is stuck

1. **Name the concept.** E.g. "this is bookmark pagination" / "this is path resolution" / "this is the chunked
   upload flow" — one sentence, so they can look it up again on their own next time.
2. **Point at the exact source.** File + method name in the installed package (`dalux_build/api/<name>.py`), and
   the relevant row in the tables above.
3. **Suggest the change in words, with a short illustrative snippet.** "In the cell that calls `list_folders`,
   swap it for `get_all_folders(..., verbose=True)` — that follows the bookmark pagination for you." Show what
   that looks like, but don't apply it to their file.
4. **Let them make the edit.** Only reach for Edit/Write yourself if they explicitly ask you to make the change.

A few recurring patterns worth steering them toward, in the same explain-first style:

- Prefer the `get_all_x` / path-based / `find_by_field` convenience methods over hand-rolled pagination or ID
  bookkeeping — that's the whole point of this client, and worth understanding *why* it exists (see "One page vs
  all pages" above) rather than just copying it in.
- If a request needs a `project_id`/`file_area_id`/`folder_id` they don't have yet, point them to
  `list_projects()` + `get_project_by_name`, `get_file_areas()` + `get_file_area_by_name`, or a path string —
  explain the resolution helper rather than handing them a raw ID to paste in.
- Any call that creates/updates/uploads data hits the user's real Dalux project. Flag this explicitly when
  explaining an upload/create/update cell, and make sure they understand it's a live side effect before they run
  it — don't just run it for them.
