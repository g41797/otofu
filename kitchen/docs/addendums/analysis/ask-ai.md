I maintain an Odin project layout where dependencies are vendored as git submodules, and I want to keep all import paths unchanged in the source code, including inside vendored dependencies.

Current setup:

- I have a main repo `my-combined` with:
  - ./src/main.odin              -- my own code
  - ./vendor/matryoshka/         -- git submodule
  - ./vendor/frame/              -- git submodule (shared dependency)

Both `src/main.odin` and `vendor/matryoshka/pipeline.odin` import the same package:

    import frame: = "frame"

I connect them at build time by running, for example:

    odin build src/main.odin -collection:frame="./vendor/frame" -collection:matryoshka="./vendor/matryoshka"

This works for regular builds and for tests run via shell scripts.

However, when I run **debugging of tests** from VS Code, I want the same “connection”:

- VS Code should see `frame` and `matryoshka` via the same `-collection:frame=./vendor/frame` and `-collection:matryoshka=./vendor/matryoshka` that the shell script uses.
- I do not want to change any import in `src/main.odin` or in `vendor/matryoshka/pipeline.odin`.
- The goal is: when I hit F5 in VS Code and debug my tests, Odin resolves `"frame"` exactly as it does from the shell.

1. What is the cleanest way to configure this for VS Code?  
   - Should I:
     - define a `tasks.json` task that wraps my build script, or  
     - put collection flags in `settings.json` / workspace settings, or  
     - modify `launch.json` directly?

2. Provide a concrete example:
   - a `scripts/build.sh` or `scripts/build-tests.sh` that builds the tests with the collections,
   - a `.vscode/tasks.json` that references that script as a pre‑launch task,
   - a `.vscode/launch.json` configuration for debugging tests, and  
   - (if supported by the Odin tooling) any relevant `settings.json` entries for `odin.buildFlags` or equivalent.

3. Assume the Odin executable is in PATH and the VS Code Odin extension exists but may not expose every Odin flag directly. Then show the most robust cross‑platform pattern (Linux / macOS / Windows).

4. If possible, include a short note for README that explains to the user:
   - “run `git submodule update --init` once”,  
   - “press F5 in VS Code to debug tests”, and  
   - “no source‑code edits are required to connect `matryoshka` and `frame`”.



                                                            ======================
                                                                GROK
                                                            ======================


The cleanest and most robust way to achieve exactly what you want (identical collection resolution for s
hell builds **and** VS Code test debugging, without touching any import statements) is:

- Use a **shell script** (or `.bat` on Windows) that centralizes the `odin test` command with all your `-collection:` flags. This keeps your build logic in one place.
- Reference that script as a **`preLaunchTask`** in `.vscode/launch.json`. This ensures that when you press **F5** (or Run → Start Debugging), VS Code first runs the script (which does the exact same build as your shell), then attaches the debugger to the resulting test binary.
- For the **Odin Language Server (ols)** — which powers IntelliSense, go-to-definition, etc. inside vendored code — configure collections via an **`ols.json`** file at the workspace root. This is the officially supported way and works independently of build flags.
- The VS Code **Odin extension** (DanielGavin.ols) does **not** have a stable `odin.buildFlags` setting that reliably passes arbitrary `-collection:` flags to every build/test invocation. Relying on `launch.json` + `preLaunchTask` is more reliable than hoping for extension-specific settings.

This pattern is fully cross-platform (Linux/macOS/Windows) when you use a simple shell script + PowerShell fallback or separate `.bat`.

### 1. Recommended files

#### `scripts/build-tests.sh` (Linux / macOS) and `scripts/build-tests.bat` (Windows)

```bash
#!/usr/bin/env bash
# scripts/build-tests.sh
# Make executable: chmod +x scripts/build-tests.sh

odin test src/main.odin \
    -collection:frame="./vendor/frame" \
    -collection:matryoshka="./vendor/matryoshka" \
    -debug \
    "$@"
```

For **Windows** (`scripts/build-tests.bat`):

```bat
@echo off
odin test src\main.odin ^
    -collection:frame="./vendor/frame" ^
    -collection:matryoshka="./vendor/matryoshka" ^
    -debug %*
```

You can add more flags (e.g., `-o:speed`, `-vet`, custom test filters via `$@` / `%*`) as needed.

#### `.vscode/tasks.json`

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build-tests",
            "type": "shell",
            "command": "${workspaceFolder}/scripts/build-tests.sh",
            "windows": {
                "command": "${workspaceFolder}/scripts/build-tests.bat"
            },
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": ["$odin"]
        }
    ]
}
```

#### `.vscode/launch.json` (for debugging tests)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Odin Tests",
            "type": "cppdbg",           // or "lldb" on macOS/Linux if you prefer
            "request": "launch",
            "program": "${workspaceFolder}/src/main.test",   // Odin test binary name
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",           // or "lldb" on macOS/Linux
            "preLaunchTask": "build-tests",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

**Notes on debugger:**
- On **Windows**: Install the **C/C++** extension (ms-vscode.cpptools) and use `"type": "cppvsdbg"` (Visual Studio Windows debugger) or `"cppdbg"` with gdb.
- On **Linux/macOS**: Use `"type": "cppdbg"` with `"MIMode": "gdb"` or `"lldb"`.
- The test binary is usually named `<package>.test` (here `main.test` if your test package is `main`). Adjust if your test file uses a different package name.

#### `ols.json` (for IntelliSense / language server — critical!)

Place this at the **root** of your workspace (`my-combined/ols.json`):

```json
{
    "$schema": "https://raw.githubusercontent.com/DanielGavin/ols/master/misc/ols.schema.json",
    "collections": [
        {
            "name": "frame",
            "path": "./vendor/frame"
        },
        {
            "name": "matryoshka",
            "path": "./vendor/matryoshka"
        }
    ]
}
```

This makes `import frame` resolve correctly inside both your code **and** inside the vendored `matryoshka` package for autocompletion, hover, etc.

(If you prefer to put it in VS Code settings, you can also configure it under `"ols.initializationOptions.collections"`, but `ols.json` is cleaner and portable.)

#### Optional: `.vscode/settings.json`

```json
{
    "files.associations": {
        "*.odin": "odin"
    },
    "odin.languageServer.path": "ols",   // if ols is in PATH
    "terminal.integrated.profiles.windows": {
        "PowerShell": {
            "source": "PowerShell",
            "icon": "terminal-powershell"
        }
    }
}
```

### 2. How to use

1. Run once: `git submodule update --init --recursive`
2. Open the project in VS Code.
3. Press **F5** → it will run the `build-tests` task (which calls your script with the exact collections) and then debug the test binary.
4. All imports (`import frame`) stay unchanged everywhere.

### 3. README note (recommended)


                                                            ======================
                                                                CHATGPT
                                                            ======================

### Development Setup

- Run `git submodule update --init --recursive` once after cloning (to fetch `vendor/frame` and `vendor/matryoshka`).
- **No source-code edits** are required to connect the shared `frame` dependency.
- Build/tests from terminal: `./scripts/build-tests.sh` (or `.bat` on Windows).
- **Debug tests in VS Code**: Just press **F5**. It automatically uses the same collection mappings as the shell script.
```

### Why this is the cleanest approach

- **Single source of truth** — the collections live only in the script (and mirrored in `ols.json` for the LSP).
- **No duplication** of flags across `tasks.json` / `launch.json`.
- Works even if the Odin VS Code extension does not expose every compiler flag directly.
- Fully cross-platform with minimal duplication (one `.sh` + one `.bat`).
- The language server and debugger both see the exact same resolution as your manual `odin test` command.

If your test entry point is not `src/main.odin` (e.g., a dedicated `tests/` directory), adjust the script and the `"program"` path accordingly. Let me know if you need a variant that runs a specific test package or passes test filters!



You’re on the right track already. The key constraint you’ve set—**no import-path changes anywhere, including vendored deps**—basically forces you into one clean solution:

> Treat `-collection` flags as part of the *build contract*, and make VS Code call the same build entrypoint as your shell.

Anything else (settings.json hacks, extension flags, etc.) will eventually drift or break.

---

# 1. What actually works cleanly

### ❌ Don’t rely on:

* VS Code Odin extension flags (not complete, not stable across setups)
* Hardcoding flags directly in `launch.json` (duplication, brittle)
* Editing imports (breaks your design)

### ✅ Do this instead:

* **Single source of truth** → `scripts/build-tests.(sh|ps1)`
* VS Code:

  * calls that script via **tasks.json**
  * debugger attaches to the produced binary via **launch.json**

This keeps:

* CLI builds ✅
* CI builds ✅
* VS Code debug ✅
  **all identical**

---

# 2. Concrete setup

## Project layout

```
my-combined/
├── src/
│   └── main.odin
├── vendor/
│   ├── frame/
│   └── matryoshka/
├── scripts/
│   ├── build-tests.sh
│   └── build-tests.ps1
├── bin/
│   └── tests            (output)
└── .vscode/
    ├── tasks.json
    ├── launch.json
    └── settings.json
```

---

# 3. Build scripts (single source of truth)

## Linux / macOS → `scripts/build-tests.sh`

```bash
#!/usr/bin/env bash
set -e

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

odin test "$ROOT_DIR/src" \
    -out:"$ROOT_DIR/bin/tests" \
    -collection:frame="$ROOT_DIR/vendor/frame" \
    -collection:matryoshka="$ROOT_DIR/vendor/matryoshka" \
    -debug
```

Make executable:

```bash
chmod +x scripts/build-tests.sh
```

---

## Windows → `scripts/build-tests.ps1`

```powershell
$Root = Resolve-Path "$PSScriptRoot\.."

odin test "$Root\src" `
    -out:"$Root\bin\tests.exe" `
    -collection:frame="$Root\vendor\frame" `
    -collection:matryoshka="$Root\vendor\matryoshka" `
    -debug
```

---

# 4. VS Code tasks.json

This just calls your script.

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build-tests (unix)",
      "type": "shell",
      "command": "${workspaceFolder}/scripts/build-tests.sh",
      "group": "build",
      "problemMatcher": []
    },
    {
      "label": "build-tests (windows)",
      "type": "shell",
      "command": "powershell",
      "args": [
        "-ExecutionPolicy",
        "Bypass",
        "-File",
        "${workspaceFolder}/scripts/build-tests.ps1"
      ],
      "group": "build",
      "problemMatcher": []
    }
  ]
}
```

---

# 5. VS Code launch.json (debugging)

Use the built binary, not Odin directly.

### Linux/macOS (lldb)

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Odin Tests (Unix)",
      "type": "lldb",
      "request": "launch",
      "program": "${workspaceFolder}/bin/tests",
      "args": [],
      "cwd": "${workspaceFolder}",
      "preLaunchTask": "build-tests (unix)"
    }
  ]
}
```

---

### Windows (cppvsdbg)

```json
{
  "name": "Debug Odin Tests (Windows)",
  "type": "cppvsdbg",
  "request": "launch",
  "program": "${workspaceFolder}/bin/tests.exe",
  "cwd": "${workspaceFolder}",
  "preLaunchTask": "build-tests (windows)"
}
```

---

# 6. settings.json (optional)

You *can* put flags here, but I don’t recommend relying on it.

If you really want IntelliSense consistency:

```json
{
  "odin.buildFlags": [
    "-collection:frame=vendor/frame",
    "-collection:matryoshka=vendor/matryoshka"
  ]
}
```

⚠️ Caveat:

* This affects editor tooling only
* It is **not guaranteed** to match actual builds
* Treat it as “nice to have”, not authoritative

---

# 7. Why this pattern is the right one

You’re effectively doing what serious systems projects do:

* **Zig** → `build.zig`
* **C/C++** → `Makefile` / `CMake`
* **Go** → `go build` (implicit contract)

You created your own:

> `scripts/build-tests.*` = your build system

That gives you:

* deterministic builds
* zero editor coupling
* identical CLI + IDE behavior
* no dependency pollution

---

# 8. Subtle but important detail

Your constraint:

```odin
import frame: = "frame"
```

means:

> `"frame"` is a logical namespace resolved **only at build time**

So your system is already **collection-driven architecture**, not path-driven.

That’s actually a strong design choice. Don’t undermine it with editor hacks.

---

# 9. README snippet (what you asked for)

You can drop this in your repo:

---

## Development Setup

### Initialize dependencies

```bash
git submodule update --init --recursive
```

---

### Running tests (CLI)

```bash
./scripts/build-tests.sh   # Linux/macOS
# or
powershell ./scripts/build-tests.ps1
```

---

### Debugging tests in VS Code

1. Open the project in VS Code
2. Press **F5**
3. Select:

   * `Debug Odin Tests (Unix)` or
   * `Debug Odin Tests (Windows)`

VS Code will:

* build tests using the same script as CLI
* automatically resolve `frame` and `matryoshka` via `-collection`
* launch the debugger

---

### Notes

* No source code changes are required to connect dependencies
* Import paths like:

```odin
import frame: = "frame"
```

work consistently across:

* CLI builds
* tests
* VS Code debugging

---

# 10. Final take

What you built is already cleaner than most Odin setups I see.

The only real rule here is:

> **Never let the editor define your build. The editor must call your build.**

If you want, next step we can push this further:

* unify build + run + bench into one script interface
* or design a mini “Odin build system” layer for your projects (like `zig build` but your style)
