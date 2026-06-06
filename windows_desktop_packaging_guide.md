# Odysseus Windows Desktop Packaging Guide

This guide details the step-by-step procedure to package Odysseus into a native, standalone Windows desktop application (`.exe`) using **PyInstaller**. 

Because Odysseus runs a FastAPI/Uvicorn backend and serves a Single Page Application (SPA) frontend, packaging it as a desktop app requires:
1. Redirection of writeable directories (databases, uploads, logs) to the user's AppData directory so that user data persists across sessions and doesn't get wiped by PyInstaller's temporary directories.
2. A lightweight GUI launcher script to start/stop the Uvicorn server and open the user's browser without a hanging terminal window.
3. PyInstaller configuration to bundle the frontend `static` assets, third-party libraries, and binary dynamic link libraries (DLLs) like the ONNX runtime.

---

## 1. Prerequisites

Before starting, ensure you have the following installed on your Windows packaging machine:
- **Python 3.11+** (matching the developer environment)
- **PyInstaller**: Install it in your active virtual environment:
  ```powershell
  pip install pyinstaller
  ```
- **Inno Setup** (Optional, recommended): A free installer compiler for Windows to bundle the directory output into a professional `.exe` setup wizard. Download from [jrsoftware.org](https://jrsoftware.org/isdl.php).

---

## 2. Code Modifications for a Packaged (Frozen) Environment

When compiled via PyInstaller, Python sets `sys.frozen = True` and extracts code and bundled files to a temporary directory in `%TEMP%` (accessible via `sys._MEIPASS`). Since this temporary directory is **read-only** and **deleted upon exit**, we must modify path constants to write to the user's local AppData directory (`%APPDATA%/Odysseus`) instead of the project root.

### Step 2.1: Update `core/constants.py`
Modify `core/constants.py` to route data and log directories to `%APPDATA%` when compiled:

```python
# core/constants.py
import os
import sys

APP_NAME = "Odysseus"
APP_VERSION = "0.9.1"

# Check if running inside a PyInstaller frozen bundle
IS_FROZEN = getattr(sys, "frozen", False)

if IS_FROZEN:
    # sys._MEIPASS holds the static files and package source code
    BASE_DIR = sys._MEIPASS + "/"
    STATIC_DIR = os.path.join(sys._MEIPASS, "static")
    
    # Writeable data/logs reside in the user's roaming profile AppData directory
    USER_DATA_ROOT = os.path.join(os.environ.get("APPDATA", os.path.expanduser("~")), APP_NAME)
    DATA_DIR = os.path.join(USER_DATA_ROOT, "data")
else:
    # Standard local developer path resolution
    BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__))) + "/"
    STATIC_DIR = os.path.join(BASE_DIR, "static")
    DATA_DIR = os.path.join(BASE_DIR, "data")

# Data file paths (dynamic paths based on the redirected DATA_DIR)
SESSIONS_FILE = os.path.join(DATA_DIR, "sessions.json")
MEMORY_FILE = os.path.join(DATA_DIR, "memory.json")
MEMORY_DOC = os.path.join(DATA_DIR, "memory_doc.md")
PERSONAL_DIR = os.path.join(DATA_DIR, "personal_docs")
RUNBOOK_DIR = os.path.join(PERSONAL_DIR, "runbook")
UPLOAD_DIR = os.path.join(DATA_DIR, "uploads")
FEATURES_FILE = os.path.join(DATA_DIR, "features.json")
SETTINGS_FILE = os.path.join(DATA_DIR, "settings.json")
```

### Step 2.2: Update `core/database.py`
Update `DATABASE_URL` in `core/database.py` to point to the redirected read-write `DATA_DIR`:

```python
# core/database.py
import os
from core.constants import DATA_DIR

# Use the resolved DATA_DIR to configure the SQLite path
DATABASE_URL = os.getenv("DATABASE_URL", f"sqlite:///{os.path.join(DATA_DIR, 'app.db')}")
```

### Step 2.3: Update `app.py` for `.env` Redirection
Modify `app.py` so that it loads `.env` from the writeable folder when frozen:

```python
# app.py
import os
import sys
from dotenv import load_dotenv

if getattr(sys, "frozen", False):
    # Load .env from %APPDATA%/Odysseus/.env
    user_app_dir = os.path.dirname(DATA_DIR) # DATA_DIR is %APPDATA%/Odysseus/data
    env_path = os.path.join(user_app_dir, ".env")
else:
    env_path = ".env"

load_dotenv(env_path, encoding="utf-8-sig")
```

---

## 3. Creating the GUI Desktop Launcher (`desktop_launcher.py`)

A windowless backend is standard for servers, but desktop users expect a clean way to start the app, open the browser, and exit. 

Create `desktop_launcher.py` in the root of your project. This script uses the standard library `tkinter` to show a clean launcher UI, starts Uvicorn on a background thread, and handles clean terminations.

```python
# desktop_launcher.py
import os
import sys
import socket
import threading
import time
import webbrowser
import tkinter as tk
from tkinter import messagebox

# Determine environment
IS_FROZEN = getattr(sys, "frozen", False)
APP_NAME = "Odysseus"

if IS_FROZEN:
    USER_DATA_ROOT = os.path.join(os.environ.get("APPDATA", os.path.expanduser("~")), APP_NAME)
    # Redirect HuggingFace/fastembed cache so they download inside AppData
    os.environ["FASTEMBED_CACHE_DIR"] = os.path.join(USER_DATA_ROOT, "models")
    os.environ["HF_HUB_DISABLE_SYMLINKS"] = "1"
    os.environ["HF_HUB_DISABLE_SYMLINKS_WARNING"] = "1"
else:
    USER_DATA_ROOT = os.path.dirname(os.path.abspath(__file__))

def get_free_port():
    """Locate a free port dynamically to prevent conflicts."""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.bind(('127.0.0.1', 0))
    port = s.getsockname()[1]
    s.close()
    return port

def run_uvicorn(port):
    """Start setup scripts and run the FastAPI server."""
    # Ensure data directory structures are populated
    import setup
    setup.create_dirs()
    setup.init_database()
    setup.create_default_admin()
    
    from app import app
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=port, log_level="info")

def wait_and_open_browser(port):
    """Wait for Uvicorn to bind to the port, then open the browser."""
    url = f"http://127.0.0.1:{port}"
    for _ in range(60):
        try:
            with socket.create_connection(("127.0.0.1", port), timeout=0.5):
                break
        except OSError:
            time.sleep(0.5)
    webbrowser.open(url)

def main():
    port = 7000
    # Fallback to free port if default is occupied
    try:
        with socket.create_connection(("127.0.0.1", port), timeout=0.2):
            port = get_free_port()
    except OSError:
        pass

    # Start FastAPI server thread
    server_thread = threading.Thread(target=run_uvicorn, args=(port,), daemon=True)
    server_thread.start()

    # Launch browser thread
    browser_thread = threading.Thread(target=wait_and_open_browser, args=(port,), daemon=True)
    browser_thread.start()

    # Build Tkinter GUI
    root = tk.Tk()
    root.title("Odysseus")
    root.geometry("360x160")
    root.resizable(False, False)
    root.configure(bg="#1E1E1E")

    # Center window
    screen_width = root.winfo_screenwidth()
    screen_height = root.winfo_screenheight()
    x = (screen_width // 2) - 180
    y = (screen_height // 2) - 80
    root.geometry(f"+{x}+{y}")

    # Title label
    label_title = tk.Label(
        root, 
        text="Odysseus Server", 
        font=("Segoe UI", 14, "bold"), 
        fg="#FFFFFF", 
        bg="#1E1E1E"
    )
    label_title.pack(pady=(15, 5))

    # Port info
    label_info = tk.Label(
        root, 
        text=f"Running at: http://127.0.0.1:{port}", 
        font=("Segoe UI", 10), 
        fg="#B0B0B0", 
        bg="#1E1E1E"
    )
    label_info.pack(pady=5)

    # Buttons frame
    btn_frame = tk.Frame(root, bg="#1E1E1E")
    btn_frame.pack(pady=15)

    def open_web():
        webbrowser.open(f"http://127.0.0.1:{port}")

    def on_quit():
        if messagebox.askokcancel("Exit Odysseus", "Are you sure you want to stop the Odysseus server?"):
            root.destroy()
            sys.exit(0)

    btn_open = tk.Button(
        btn_frame, 
        text="Open Browser", 
        command=open_web,
        font=("Segoe UI", 10), 
        fg="#FFFFFF", 
        bg="#2D5A27",
        activebackground="#3E7C36",
        activeforeground="#FFFFFF",
        relief=tk.FLAT,
        width=12
    )
    btn_open.grid(row=0, column=0, padx=10)

    btn_stop = tk.Button(
        btn_frame, 
        text="Stop Server", 
        command=on_quit,
        font=("Segoe UI", 10), 
        fg="#FFFFFF", 
        bg="#8B2500",
        activebackground="#A62C00",
        activeforeground="#FFFFFF",
        relief=tk.FLAT,
        width=12
    )
    btn_stop.grid(row=0, column=1, padx=10)

    root.protocol("WM_DELETE_WINDOW", on_quit)
    root.mainloop()

if __name__ == "__main__":
    main()
```

---

## 4. PyInstaller Packaging Commands

We recommend packaging the application as a directory structure (`--onedir`) rather than a single massive file (`--onefile`). 
- **`--onedir` (Recommended)**: Loads significantly faster on startup because it does not need to extract heavy packages (like PyTorch, Fastembed, NumPy, and ONNX DLLs) to a temporary location on every run.
- **`--onefile`**: Creates a single `.exe` but introduces a 5-10 second delay on launch.

Run the following command in PowerShell to build:

```powershell
pyinstaller `
  --onedir `
  --noconsole `
  --name "Odysseus" `
  --add-data "static;static" `
  --hidden-import "uvicorn.loops" `
  --hidden-import "uvicorn.loops.auto" `
  --hidden-import "uvicorn.protocols" `
  --hidden-import "uvicorn.protocols.http" `
  --hidden-import "uvicorn.protocols.http.auto" `
  --hidden-import "uvicorn.protocols.websockets" `
  --hidden-import "uvicorn.protocols.websockets.auto" `
  --hidden-import "uvicorn.lifespan" `
  --hidden-import "uvicorn.lifespan.on" `
  --collect-all "fastembed" `
  desktop_launcher.py
```

### Parameter Breakdown:
* `--onedir`: Generates a folder (`dist/Odysseus`) containing all executable binaries.
* `--noconsole`: Prevents the Windows CLI console window from appearing in the background.
* `--add-data "static;static"`: Directs PyInstaller to bundle the static folder containing the frontend SPA.
* `--hidden-import "..."`: Instructs PyInstaller to build dynamic imports used by Uvicorn's protocol lifespans that static analysis tools miss.
* `--collect-all "fastembed"`: Ensures all metadata, ONNX dependencies, and models for the embedding backend are located and bundled.

After execution completes, your build output resides in: `dist/Odysseus/Odysseus.exe`.

---

## 5. Compiling a Professional Setup Wizard (Inno Setup)

To distribute Odysseus as a standard Windows installation program, you can compile the `dist/Odysseus` output directory into an installer.

Create `installer.iss` in the project folder and compile it in the Inno Setup compiler application:

```ini
; installer.iss
[Setup]
AppName=Odysseus
AppVersion=1.0.0
DefaultDirName={userappdata}\OdysseusApp
DefaultGroupName=Odysseus
OutputDir=dist-setup
OutputBaseFilename=OdysseusInstaller
Compression=lzma
SolidCompression=yes
DisableDirPage=yes
PrivilegesRequired=lowest

[Tasks]
Name: "desktopicon"; Description: "{cm:CreateDesktopIcon}"; GroupDescription: "{cm:AdditionalIcons}"; Flags: unchecked

[Files]
Source: "dist\Odysseus\*"; DestDir: "{app}"; Flags: recursesubdirs createallsubdirs

[Icons]
Name: "{group}\Odysseus"; Filename: "{app}\Odysseus.exe"
Name: "{userdesktop}\Odysseus"; Filename: "{app}\Odysseus.exe"; Tasks: desktopicon

[Run]
Filename: "{app}\Odysseus.exe"; Description: "Launch Odysseus"; Flags: postinstall nowait
```

Running this compiler produces `dist-setup/OdysseusInstaller.exe`, which handles installation shortcuts and clean uninstalls on user machines.

---

## 6. Verification Checklist

To confirm the packaged application is working as expected:
1. **Launch**: Locate and run `dist/Odysseus/Odysseus.exe`. Ensure the Tkinter GUI appears and the web browser automatically launches pointing to `http://127.0.0.1:7000`.
2. **First Run Setup**: Verify that you are prompted for login credentials and that the default admin is created inside `%APPDATA%/Odysseus/data/auth.json`.
3. **Data Persistence**:
   - Create a chat session or save a setting.
   - Close the application window (or click "Stop Server").
   - Reopen `Odysseus.exe` and confirm that all previously created sessions and settings persist.
4. **ONNX RAG Validation**: Send a query that requires document embeddings. Ensure the fastembed library correctly initializes the ONNX runtime from the bundled files and creates the vector database under `%APPDATA%/Odysseus/data/chroma`.
