# Plan: Odin Dev Environment — Repo + Toolchain + Hello Tofu

## Context

The Odin port is the primary language port target (LANGUAGE_PORT_FRAMEWORK.md).
Before architecture mapping or idiom documents can be written meaningfully, a working Odin
dev environment must exist. Real mapping work comes from writing real Odin code.

This plan bootstraps that environment:
- A new, separate GitHub repo for the Odin implementation (`otofu`)
- Odin toolchain installed and IDE ready
- A "hello tofu" program that runs locally and passes CI on GitHub

The name follows the `pytofu` pattern used for the Python port planning.

---

## Deliverables

| Deliverable | Who creates it | Where |
|---|---|---|
| Repo skeleton (structure, README, .gitignore, build) | Claude Code | `/home/g41797/dev/root/github.com/g41797/otofu/` |
| `hello_tofu.odin` — the hello tofu program | Claude Code | `otofu/src/` |
| GitHub Actions CI workflow | Claude Code | `otofu/.github/workflows/` |
| Odin compiler installation | User | local machine |
| IDE setup (VS Code + OLS) | User | local machine |
| GitHub repo creation + first push | User | github.com/g41797/otofu |

---

## Step 1 — Repo Skeleton

Claude Code creates the directory structure locally at
`/home/g41797/dev/root/github.com/g41797/otofu/`.

### Directory layout

```
otofu/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
│   └── hello_tofu.odin
├── .gitignore
└── README.md
```

### `.gitignore`
Standard Odin ignores: compiled binaries, `*.exe`, `*.pdb`, `*.o`.

### `README.md`
One paragraph: what this repo is, what it ports, link to tofusite.

---

## Step 2 — Hello Tofu Program

### What it must demonstrate

"Hello tofu" is not a language hello world. It is the smallest possible
proof that the Odin environment can express tofu's core data structures correctly.

It must:
1. Define `BinaryHeader` as a packed struct (16 bytes, matching wire format).
2. Create an instance with `HelloRequest` opcode (value 3).
3. Assert `size_of(BinaryHeader) == 16` at compile time.
4. Print field values at runtime.

### Why this specific program

`BinaryHeader` is the foundation of the entire protocol.
If packed structs and bit fields work correctly in Odin, the port is feasible at the data layer.
If the size assert fails, the port has a fundamental problem that must be resolved before any further work.

### BinaryHeader fields (from MESSAGES-foundations.md)

| Field | Type | Size |
|---|---|---|
| channel_number | u16, little-endian | 2 bytes |
| proto (opcode 4 bits, origin 1 bit, more 1 bit, reserved 2 bits) | u8 | 1 byte |
| status | u8 | 1 byte |
| message_id | u64, little-endian | 8 bytes |
| thl (text headers length) | u16, little-endian | 2 bytes |
| bl (body length) | u16, little-endian | 2 bytes |
| **total** | | **16 bytes** |

### OpCodes needed (from MESSAGES-protocol-operations.md)

Only `HelloRequest = 3` is needed for the hello program.
Define the full enum for completeness (10 values).

---

## Step 3 — GitHub Actions CI

### Trigger
Push to `main`. Pull requests to `main`.

### Steps
1. `actions/checkout`
2. Install Odin compiler (official release, latest stable, Linux amd64).
3. Build: `odin build src/ -out:hello_tofu`.
4. Run: `./hello_tofu`. Exit code 0 = pass.

### Odin CI install method
Download the official release archive from `github.com/odin-lang/Odin/releases`.
Extract to a known path. Add to `PATH`.
No package manager dependency (avoids version drift between local and CI).

---

## Step 4 — User Actions (after Claude Code creates the files)

In order:
1. Install Odin compiler locally (same version as CI).
   Reference: https://odin-lang.org/docs/install/
2. Install VS Code extension: `jBugman.odin-lang` (or current recommended).
   Install OLS (Odin Language Server) for autocomplete and error checking.
3. Create GitHub repo `g41797/otofu` (public, empty, no README — Claude Code provides it).
4. `cd /home/g41797/dev/root/github.com/g41797/otofu`
5. `git init && git remote add origin git@github.com:g41797/otofu.git`
6. `git add . && git commit -m "Initial: hello tofu skeleton"`
7. `git push -u origin main`
8. Verify GitHub Actions run passes.

---

## Step 5 — Record in tofusite

After the first CI pass, add one entry to `root/consolidation/SITEMAP.md`:
- `github.com/g41797/otofu` — Odin implementation repo. Status: bootstrapped.

Update `root/consolidation/CHECKPOINT.md`:
- Record that the Odin dev environment is live.
- Record the Odin compiler version in use (for reproducibility).

---

## Critical File Paths

Files Claude Code will create:
- `/home/g41797/dev/root/github.com/g41797/otofu/src/hello_tofu.odin`
- `/home/g41797/dev/root/github.com/g41797/otofu/.github/workflows/ci.yml`
- `/home/g41797/dev/root/github.com/g41797/otofu/.gitignore`
- `/home/g41797/dev/root/github.com/g41797/otofu/README.md`

Files updated in tofusite after CI passes:
- `root/consolidation/SITEMAP.md`
- `root/consolidation/CHECKPOINT.md`

---

## Verification (Done State)

All three must be true:
1. `odin build src/ -out:hello_tofu` completes with no errors locally.
2. `./hello_tofu` prints field values and exits 0.
3. GitHub Actions badge is green on `main`.

---

## What Comes Next (After This Plan)

Once the environment is live:
- Write a minimal non-blocking TCP PoC in Odin (answers ODIN-RESEARCH.md open questions).
- From real Odin experience, write the Zig→Odin mapping document.
- Mapping covers: error handling, memory/allocators, packed structs, module structure,
  thread coordination, and tofu-specific idioms.
