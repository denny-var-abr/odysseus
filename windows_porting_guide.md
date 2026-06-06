# Odysseus Windows Porting & Compatibility Guide

This document catalogs the platform discrepancies, root causes, and architectural changes required to run Odysseus natively and robustly on Windows.

---

## 1. Active Workspace Path Resolution
### The Problem
When the user sets an active workspace (e.g., `c:\Repo\odysseus`), the AI agent frequently uses path formatting like `"/foo.txt"`, `"/workspace/foo.txt"`, or absolute drive-relative patterns like `"/c:/Repo/odysseus/foo.txt"`. 
- **`os.path.isabs` behavior on Windows**: Python's `ntpath.isabs` evaluates `/foo.txt` as `True` because it starts with a separator. This prevents the resolver from prefixing the active workspace path, resolving it to `C:\foo.txt` (which is outside the workspace).
- **Drive-relative ValueError**: `os.path.realpath("/c:/Repo/odysseus/foo.txt")` resolves to `c:Repo\odysseus\foo.txt` (missing the root backslash). When passed to `os.path.commonpath([resolved, base])` alongside the drive-absolute workspace path `C:\Repo\odysseus`, it throws:
  `ValueError: Can't mix absolute and relative paths`
  This exception is caught and reported back as `"outside the workspace"`.

### The Solution
Update `_resolve_tool_path_in_workspace` in `src/tool_execution.py` to:
1. Strip cygwin-style prefixes (e.g., `/c/` or `/C/`) and leading slash prefixes on Windows drive letters (e.g., `/c:` or `/C:`).
2. Strip literal `workspace/` or `/workspace/` prefixes.
3. Clean and strip leading slashes/backslashes (`lstrip('/\\')`) to force the path to resolve relatively under the workspace root path as a first pass.
4. Fall back to resolving as a standard absolute path, verifying that it is nested under the normalized workspace folder.

---

## 2. Temporary Path Resolution
### The Problem
During path confinement checks (in `_resolve_tool_path`), temporary directory paths are rejected as `"outside the allowed roots"`.
- **`TMPDIR` Environment Variable**: `_tool_path_roots()` queries `os.environ.get("TMPDIR")`. However, Windows uses `TEMP` or `TMP` instead of `TMPDIR`.

### The Solution
Modify `_tool_path_roots()` in `src/tool_execution.py` to search for `TEMP` or `TMP` environment variables on Windows:
```python
tmpdir = os.environ.get("TMPDIR") or os.environ.get("TEMP") or os.environ.get("TMP")
```

---

## 3. Sensitive Path Checks & Path Separators
### The Problem
`_is_sensitive_path` fails to identify sensitive paths (like `.ssh/authorized_keys` or `.gnupg/pubring.kbx`) if they contain forward slashes on Windows.
- **Separator splitting**: The function performs `resolved.split(os.sep)`. On Windows, `os.sep` is `\`. If the path is `/home/user/.ssh/authorized_keys`, the split returns a single item list, bypassing the checks for `.ssh` or `.gnupg`.

### The Solution
Normalize path separators using `os.path.normpath` or replace all forward slashes with `os.sep` before checking:
```python
normalized = resolved.replace("/", os.sep)
parts = normalized.split(os.sep)
```

---

## 4. Subprocess Shell Execution
### The Problem
The agent's `bash` tool invokes `asyncio.create_subprocess_shell` directly. On Windows, this uses `cmd.exe` by default. Since agent models output Unix shell/bash syntax, simple commands fail to run.

### The Solution
Leverage the `find_bash()` helper from `core/platform_compat.py`. If running on Windows and a bash executable is found, invoke it explicitly:
```python
from core.platform_compat import find_bash, IS_WINDOWS

bash_bin = find_bash() if IS_WINDOWS else None
if bash_bin:
    proc = await asyncio.create_subprocess_exec(
        bash_bin, "-c", content,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        env=_subproc_env,
        cwd=workspace or _AGENT_WORKDIR,
    )
else:
    # Fallback to default shell
    ...
```

---

## 5. ESM Loader Scheme Rejections in Tests
### The Problem
Several JavaScript tests running under Node.js fail with:
`Error [ERR_UNSUPPORTED_ESM_URL_SCHEME]: Only URLs with a scheme in: file, data, and node are supported by the default ESM loader. On Windows, absolute paths must be valid file:// URLs.`
This happens because Node is executed with dynamic JavaScript containing absolute drive paths (e.g., `import { ... } from 'C:/Repo/odysseus/...'`).

### The Solution
Convert paths to valid `file://` URLs on Windows when generating JavaScript modules for Node.js subprocesses in tests:
```python
from urllib.request import pathname2url
file_url = f"file:///{pathname2url(path)}"
```

---

## 6. Concurrent File Operations
### The Problem
On Windows, concurrent file replacements (`os.replace`) fail with `PermissionError: [WinError 5] Access is denied` if another file handle is open.

### The Solution
Ensure file handles are closed immediately using context managers, or implement a file-locking or retry mechanism inside `core/atomic_io.py` to gracefully wait for handles to release.
