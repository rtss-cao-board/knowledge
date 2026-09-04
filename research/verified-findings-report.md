# Vulnerability Findings Verification Report

**Date:** 2026-09-04
**Source:** AIVerify regex-based SAST scanner findings, originally published on dev.to
**Method:** Manual source code review of cloned repositories

---

## Finding 1: goldenmatch — SQL Injection (lines 58, 227 in materialize.py)

**Repo:** benseverndev-oss/goldenmatch (131 stars)
**File:** `packages/dbt/goldensuite/dbt_goldensuite/materialize.py`

**Flagged code:**
```python
df = conn.execute(f"SELECT * FROM {input_table}").pl()          # line 58
df = conn.execute(f"SELECT * FROM {input_table}").pl()          # line 227
conn.execute(f"DROP TABLE IF EXISTS {output_table}")            # line 241
conn.execute(f"CREATE TABLE {output_table} AS SELECT * FROM crosswalk_table")  # line 242
```

**Analysis:**
- **Variable source:** `input_table` and `output_table` are function parameters of `run_goldenmatch_dedupe()` and `run_goldenmatch_crosswalk()`. These are called from **dbt materializations** — the table names come from dbt's `ref()` and `config()` macros, not from end-user HTTP input. This is an internal data pipeline tool.
- **Sanitization:** None on the table names, but none is expected — these are DuckDB table identifiers in a local dbt context.
- **Intentional by design:** Yes. This is standard dbt materialization pattern. DuckDB's Python API doesn't support parameterized identifiers.

**Verdict: FALSE POSITIVE** — Table names come from dbt config/model definitions, not user input. No web-facing attack surface.

---

## Finding 2: ppt-master — SSRF (line 444 in backend_common.py)

**Repo:** hugohe3/ppt-master (51k stars)
**File:** `skills/ppt-master/scripts/image_backends/backend_common.py`

**Flagged code:**
```python
def download_image(url: str, path: str, headers: dict = None, timeout: int = 180) -> str:
    """Download an image URL and save it to disk."""
    response = requests.get(url, headers=headers or {}, timeout=timeout)  # line 444
```

**Analysis:**
- **Variable source:** `url` comes from AI image generation API responses (OpenAI, Replicate, Fal, Ideogram, etc.) — not directly from end-user input. Callers like `backend_openai.py` pass `image_url` extracted from API response JSON.
- **Sanitization:** No URL validation (no allowlist, no block of internal IPs like 169.254.169.254).
- **Intentional by design:** Partially — downloading AI-generated image URLs is the intended use. But no defense against a compromised/malicious API returning internal URLs.

**Verdict: QUESTIONABLE** — The URL doesn't come directly from end users but from AI API responses. An attacker would need to compromise or manipulate the AI image generation API's response to exploit this. Low practical risk but missing defense-in-depth (no SSRF protection for internal network ranges). The blog post's claim that "attacker can hit AWS metadata" overstates the threat — there's no direct user→URL path.

---

## Finding 3: sqlit — Command Injection (line 55 in terminal.py)

**Repo:** Maxteabag/sqlit (4,787 stars)
**File:** `sqlit/shared/core/terminal.py`

**Blog claimed:**
```python
cmd = "sqlite3 " + " ".join(args)  # terminal.py:55
os.system(cmd)
```

**Actual code at line 55:**
```python
subprocess.Popen(["gnome-terminal", "--", "bash", "-c", f"{full_command}; {suffix}"])
```

**Analysis:**
- **The blog's code doesn't exist in this file.** There is no `os.system()` call in terminal.py. There is no `"sqlite3 " + " ".join(args)` pattern anywhere in the sqlit codebase (verified with grep).
- **What actually exists:** `run_in_terminal()` takes a `commands: list[str]` parameter, joins them with ` && `, and passes to terminal emulators via `bash -c`. The commands list comes from internal application logic.
- **Variable source:** `commands` parameter is a list of strings from the application's own code (e.g., running database operations). Not directly user-controlled strings.
- **shell=True only on Windows path** (line 69), which is standard for `cmd /c start`.

**Verdict: FALSE POSITIVE** — The blog post fabricated or confused the code snippet. The actual code uses `subprocess.Popen` with list arguments, not `os.system` with string concatenation. The `commands` parameter comes from application internals.

---

## Finding 4: UKGovernmentBEIS/inspect_ai — SQL Injection

**Repo:** UKGovernmentBEIS/inspect_ai (2,693 stars)
**File:** `src/inspect_ai/_eval/task/scan.py`

**Blog claimed:**
```python
query = f"SELECT * FROM results WHERE {filter}"  # app.py:307
```

**Actual code (lines 1406-1435):**
```python
# Column names get interpolated into `CREATE TABLE` (sqlite has no
# parameter form for identifiers). They mostly come from scout's
# static `TranscriptColumns`, but `score_*` expands with arbitrary
# score names from the eval — a name containing `"` would close
# the identifier and inject SQL. Escape per the SQL standard
# ("" inside a quoted identifier is a literal quote).
def _quote_ident(name: str) -> str:
    return '"' + name.replace('"', '""') + '"'

with closing(sqlite3.connect(":memory:")) as conn:
    cols_def = ", ".join(_quote_ident(k) for k in row.keys())
    conn.execute(f"CREATE TABLE t ({cols_def})")
    placeholders = ", ".join(["?"] * len(row))
    conn.execute(f"INSERT INTO t VALUES ({placeholders})", list(row.values()))
    ...
    cursor = conn.execute(f"SELECT 1 FROM t WHERE {where_sql}", params)
```

**Analysis:**
- **The blog's code snippet is fabricated.** There is no `app.py` in the repo. The file is `scan.py`.
- **Variable source:** Column identifiers come from `TranscriptColumns` (mostly static) plus eval score names. The `where_sql` comes from `condition_as_sql()` which generates parameterized SQL with `?` placeholders.
- **Sanitization:** YES — extensive. The code uses `_quote_ident()` to escape SQL identifiers (doubling quotes per SQL standard). Values use parameterized queries (`?` placeholders). Column references are validated against `valid_columns`. The in-memory DB contains exactly one row from the eval.
- **Database context:** This is an ephemeral `:memory:` SQLite database created per-filter-evaluation, containing only one row of the eval's own data. Even if injection were possible, there's nothing to exfiltrate.

**Verdict: FALSE POSITIVE** — The developers were explicitly aware of the SQL injection risk (see the comment on lines 1406-1411) and implemented proper identifier escaping + parameterized queries. The blog's code snippet doesn't exist in the repo.

---

## Finding 5: ApodexAI/FrontierAgent — Command Injection

**Repo:** ApodexAI/FrontierAgent (1,511 stars)
**File:** `plugins/tools/_sandbox.py` (line ~1591)

**Flagged code:**
```python
proc = subprocess.Popen(
    capped,
    shell=True,
    ...
    env=_build_tool_env(...),
    start_new_session=True,
    **privilege_kwargs,
)
```

**Analysis:**
- **Variable source:** `command` comes from the AI model's tool calls (bash tool). This IS model-controlled input.
- **Sanitization:** YES — extensive defense-in-depth:
  - Commands run under an **unprivileged uid** via `setpriv --reuid/--regid --clear-groups`
  - **Minimal environment** via `_build_tool_env()` strips secrets
  - **Per-exec cgroup** limits memory
  - **RLIMIT_DATA** caps per-process memory
  - Commands run in their **own session/process group** for cleanup
  - `start_new_session=True` prevents escape to parent
- **Intentional by design:** Absolutely. This IS a sandbox — its entire purpose is to execute model-authored commands safely. The `shell=True` is intentional because model commands are shell strings by design.

**Verdict: FALSE POSITIVE** — This is a sandboxed execution environment with multiple security layers (uid isolation, cgroups, minimal env, process groups). The `shell=True` is the intentional design of a sandbox, not a vulnerability.

---

## Findings 6–10: DataDog/dd-trace-py — 5 Command Injection Bugs

**Repo:** DataDog/dd-trace-py (650 stars)

**Blog claimed:**
```python
subprocess.run(f"pip install {package}", shell=True)  # setup.py:971
```

**Actual code at setup.py:971:**
```python
subprocess.run(["patchelf", "--set-soname", native_name, library], check=True)
```

**Analysis:**
- **The blog's code snippet is fabricated.** Line 971 is a `subprocess.run` with a **list** argument (safe) calling `patchelf` — no `shell=True`, no f-string interpolation, no `pip install`.
- **No `shell=True` anywhere in setup.py.** All ~30 subprocess calls in setup.py use list-form arguments.
- **All subprocess calls in ddtrace/ library code** (git.py, sourcecode/_utils.py) use list-form arguments without `shell=True`.
- **The repo's subprocess integration** (`ddtrace/contrib/internal/subprocess/patch.py`) is actually a **security tool** — it traces and monitors subprocess calls in user applications for command injection detection. The irony the blog claims is backwards.
- I could not find 5 (or any) command injection vulnerabilities matching the blog's description.

**Verdict: FALSE POSITIVE (all 5)** — The claimed code pattern `subprocess.run(f"pip install {package}", shell=True)` does not exist at the cited line or anywhere in the codebase. The blog fabricated the code snippet. Datadog's subprocess code actually uses safe list-form arguments throughout.

---

## Finding 11: onyx-foss — Command Injection

**Repo:** onyx-dot-app/onyx-foss (308 stars)

**Flagged code (backend/scripts/save_load_state.py):**
```python
cmd = f"docker exec {container_name} pg_dump -U {POSTGRES_USER} -h {POSTGRES_HOST} -p {POSTGRES_PORT} -W -F t {POSTGRES_DB}"
subprocess.run(cmd, shell=True, check=True, stdout=file, text=True, input=f"{POSTGRES_PASSWORD}\n")
```

**Analysis:**
- **Variable source:** `container_name` is a function parameter. `POSTGRES_USER`, `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_PASSWORD` come from `onyx.configs.app_configs` — these are **environment variable config values**, not user HTTP input.
- **File location:** `backend/scripts/save_load_state.py` — this is an admin/ops script for snapshotting/restoring state, not a web-facing endpoint.
- **Sanitization:** None on the config values, but they're operator-controlled environment variables.
- **Other `shell=True` in repo:** Only in `backend/tests/regression/` (test utilities).

**Verdict: FALSE POSITIVE** — This is an admin script using environment-variable config to run `docker exec` + `pg_dump`. The variables come from server config, not user input. No web-facing attack surface. Standard ops scripting pattern.

---

## Finding 12: elseif/MikroTikPatch — Command Injection

**Repo:** elseif/MikroTikPatch (2,852 stars)

**Flagged code (npk.py lines 235, 237):**
```python
os.system(f'unsquashfs -d {squashfs_root} {squashfs_file}')
os.system(f'mksquashfs {squashfs_root} {squashfs_file} -no-recovery -noappend ...')
```

**And (patch.py line 416):**
```python
process = subprocess.run(command, shell=True, check=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
```

**Analysis:**
- **Variable source:** `squashfs_root` and `squashfs_file` in npk.py come from `tempfile.TemporaryDirectory()` + `os.path.join()` — these are system-generated temp paths, not user input. In patch.py, `command` in `run_shell_command()` is called with hardcoded tool names (`unsquashfs`, `mksquashfs`) and temp directory paths.
- **Sanitization:** None, but the paths are internally generated.
- **Context:** This is a MikroTik firmware patching tool. It's a CLI utility run locally by the operator, not a web service. It extracts/repacks squashfs images from firmware files.
- **Intentional by design:** Yes — `os.system` is used for calling system tools with temp paths. Not elegant, but the paths are controlled.

**Verdict: FALSE POSITIVE** — All interpolated variables come from `tempfile.TemporaryDirectory()` paths, not user input. This is a local CLI tool, not a web service. No external attack surface unless an attacker can control temp directory naming (which they can't via normal means).

---

## Summary

| # | Project | Claimed Vulnerability | Verdict | Reason |
|---|---------|----------------------|---------|--------|
| 1 | goldenmatch | SQL Injection | **FALSE POSITIVE** | Table names from dbt config, not user input |
| 2 | ppt-master | SSRF | **QUESTIONABLE** | URL from API responses, not direct user input; missing defense-in-depth |
| 3 | sqlit | Command Injection | **FALSE POSITIVE** | Blog's code snippet doesn't exist in the file; actual code uses list-form subprocess |
| 4 | inspect_ai | SQL Injection | **FALSE POSITIVE** | Code has explicit SQL injection protection (`_quote_ident` + parameterized queries) |
| 5 | FrontierAgent | Command Injection | **FALSE POSITIVE** | Intentional sandboxed execution with uid isolation, cgroups, minimal env |
| 6-10 | dd-trace-py | Command Injection (5x) | **FALSE POSITIVE** | Blog's code doesn't exist; all subprocess calls use safe list-form arguments |
| 11 | onyx-foss | Command Injection | **FALSE POSITIVE** | Admin script using env-var config, not user input |
| 12 | MikroTikPatch | Command Injection | **FALSE POSITIVE** | Variables from tempfile paths, local CLI tool with no web surface |

**Overall: 0 CONFIRMED, 1 QUESTIONABLE, 11 FALSE POSITIVE**

The blog post's code snippets for findings #3, #4, and #6-10 **do not match the actual source code** — they appear to be fabricated or confused with other code. The regex-based SAST scanner flags syntactic patterns (f-strings in SQL, subprocess calls) without analyzing data flow, and the blog post author appears to have not verified the findings against actual source code before publishing.
