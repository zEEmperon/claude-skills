---
name: dotnet-project-creator
description: >
  Create a new .NET Core solution with one project inside it using the dotnet SDK CLI.
  Supports all .NET-compliant languages: C#, F#, and VB.
  Use this skill whenever the user wants to scaffold, initialize, bootstrap, or set up a .NET, C#, F#, VB,
  or ASP.NET Core project — even if they say things like "create a new C# project", "init a .NET solution",
  "scaffold a webapi", "start a new F# console app", etc. Also triggers for requests about Directory.Build.props,
  Directory.Packages.props, or centralized NuGet configuration in a new project context.
---

# .NET Project Creator (C#, F#, VB)

Scaffold a .NET solution + project using only the `dotnet` CLI. Supports C#, F#, and VB. No files are created manually where the SDK can do it.

---

## Step 1 — Gather Information from the User

Ask for **everything in a single message**. To do this, run `dotnet new list --type project` first (without a language filter), then present the full output alongside all questions. This avoids a back-and-forth conversation.

1. Run `dotnet new list --type project` and include the **complete, unfiltered, raw** CLI output in your message. Do NOT reformat into a different table, do NOT truncate rows, do NOT add "(+ more …)" summaries, do NOT omit any entries. Copy-paste the full terminal output as-is.
2. Then ask for all of the following **in the same message**:
   1. **Target directory** — full path where the solution will be created. Must exist and be empty (config/metadata files like `.git/`, `.gitignore`, `.gitattributes`, `.editorconfig`, `.claude/`, `.vscode/` are allowed and will not be touched).
   2. **Solution name** — e.g. `MyApp`
   3. **Project name** — e.g. `MyApp.Api`
   4. **Language** — `C#` (default), `F#`, or `VB`. Note: not all templates are available for every language.
   5. **Project template** — pick a short name from the list above. Note that some templates are only available for certain languages (see the Language column).
   6. **Target framework** (optional) — e.g. `net9.0`, `net8.0`. Leave blank to use the SDK default.
   7. **Directory.Build.props** — yes/no. Centralizes shared MSBuild properties (TreatWarningsAsErrors, Nullable, etc.) across all projects in the solution.
   8. **Directory.Packages.props** (Central Package Management) — yes/no. Only relevant if the user plans multiple projects or wants to centralize NuGet versions.

The user should be able to answer all 8 items in a single reply. Do **not** split this into multiple rounds of questions.

> **Never skip asking for the target directory.** Refuse to proceed without it.

---

## Step 2 — Pre-flight Checks

Before running any `dotnet` commands, validate:

**Windows (PowerShell):**
```powershell
# 1. dotnet SDK present
dotnet --version

# 2. Directory exists
Test-Path "<TARGET_DIR>" -PathType Container

# 3. Directory contents (allow config/metadata: .git, .gitignore, .gitattributes, .editorconfig, .claude, .vscode)
Get-ChildItem "<TARGET_DIR>" -Force | Where-Object { $_.Name -notin '.git','.gitignore','.gitattributes','.editorconfig','.claude','.vscode' }
```

**Linux / macOS (bash):**
```bash
# 1. dotnet SDK present
dotnet --version

# 2. Directory exists
test -d "<TARGET_DIR>" && echo "exists" || echo "missing"

# 3. Directory contents (allow config/metadata)
find "<TARGET_DIR>" -mindepth 1 -maxdepth 1 ! -name '.git' ! -name '.gitignore' ! -name '.gitattributes' ! -name '.editorconfig' ! -name '.claude' ! -name '.vscode' | head -5
```

**Rules:**
- If `dotnet` is missing → tell the user to install the .NET SDK from https://dotnet.microsoft.com/download and stop.
- If directory is missing → create it (`New-Item -ItemType Directory` on Windows, `mkdir -p` on Linux/macOS) and inform the user.
- If directory has unexpected files (anything beyond the allowed config/metadata list above) → stop and warn; do **not** overwrite.

---

## Step 3 — Scaffold the Solution

Always use **absolute paths** — do not rely on `cd` because the working directory may reset between terminal invocations.

```bash
dotnet new sln --name "<SOLUTION_NAME>" --output "<TARGET_DIR>"
```

> `--output "<TARGET_DIR>"` keeps the solution file in the target directory (not a subdirectory).

> **Important:** .NET 9+ SDK creates `.slnx` (XML-based) by default instead of `.sln`. After this command, check which file was created (`.sln` or `.slnx`) and use that filename in all subsequent commands. Do not hardcode `.sln`.

---

## Step 4 — Scaffold the Project

Use absolute paths for `--output`:

```bash
# Basic (no framework specified)
dotnet new <TEMPLATE> --name "<PROJECT_NAME>" --output "<TARGET_DIR>/<PROJECT_NAME>" --language "<LANGUAGE>"

# With explicit framework
dotnet new <TEMPLATE> --name "<PROJECT_NAME>" --output "<TARGET_DIR>/<PROJECT_NAME>" --language "<LANGUAGE>" --framework <FRAMEWORK>
```

The project lands in `<TARGET_DIR>/<PROJECT_NAME>/`.

---

## Step 5 — Add Project to Solution

Use the actual solution filename discovered in Step 3 (`.sln` or `.slnx`). Use the correct project file extension for the chosen language:
- C# → `.csproj`
- F# → `.fsproj`
- VB → `.vbproj`

```bash
dotnet sln "<TARGET_DIR>/<SOLUTION_FILE>" add "<TARGET_DIR>/<PROJECT_NAME>/<PROJECT_NAME>.<EXT>"
```

Where `<SOLUTION_FILE>` is the actual file (e.g. `MyApp.slnx` or `MyApp.sln`) and `<EXT>` is `csproj`, `fsproj`, or `vbproj`.

---

## Step 6 — Optional: Directory.Build.props

If the user chose yes, create `<TARGET_DIR>/Directory.Build.props`. Tailor the content to the chosen language:

**C# (full set of properties):**
```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
    <ImplicitUsings>enable</ImplicitUsings>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

**F# or VB (skip C#-only properties):**
```xml
<Project>
  <PropertyGroup>
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

> `Nullable` and `ImplicitUsings` are C#-only and must not be added for F# or VB projects.

> **Note for Linux**: The filename is case-sensitive. It **must** be `Directory.Build.props` (capital D, B, p).

---

## Step 7 — Optional: Directory.Packages.props (Central Package Management)

If the user chose yes, create `<TARGET_DIR>/Directory.Packages.props`:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
  </ItemGroup>
</Project>
```

When `Directory.Packages.props` is created, also strip the `Version` attribute from any `<PackageReference>` items generated by the project template (if any). Use `str_replace` or `sed` to patch the `.csproj`.

---

## Step 8 — Verify the Result

**Windows (PowerShell):**
```powershell
# Show final tree
Get-ChildItem -Recurse "<TARGET_DIR>" | Where-Object { $_.FullName -notmatch '\\.git\\' -and $_.FullName -notmatch '\\.claude\\' }

# Build to confirm everything compiles (do NOT use --no-restore; templates may not auto-restore)
dotnet build "<TARGET_DIR>" -v minimal
```

**Linux / macOS (bash):**
```bash
# Show final tree
find "<TARGET_DIR>" -not -path '*/.git/*' | sort

# Build to confirm everything compiles
dotnet build "<TARGET_DIR>" -v minimal
```

> Use `dotnet build` **without** `--no-restore` — not all templates auto-restore packages during creation.

---

## Step 9 — Present the Result

Report:
- ✅ Files created (solution, project, optional props files)
- 📁 Directory tree
- 🔨 Build status (pass/fail)
- Next steps the user can take (e.g., `dotnet run`, adding packages with `dotnet add package <Name>`)

---

## Cleanup on Failure

If **any** step fails, roll back everything created in this run. Track which artifacts were created.

**Windows (PowerShell):**
```powershell
# If the target directory itself was created by this run, remove it entirely
Remove-Item -Recurse -Force "<TARGET_DIR>"

# If the target directory already existed, remove only what was added:
Remove-Item -Force "<TARGET_DIR>/<SOLUTION_FILE>"          # .sln or .slnx
Remove-Item -Recurse -Force "<TARGET_DIR>/<PROJECT_NAME>"
Remove-Item -Force "<TARGET_DIR>/Directory.Build.props"    # if created
Remove-Item -Force "<TARGET_DIR>/Directory.Packages.props" # if created
```

**Linux / macOS (bash):**
```bash
# If the target directory itself was created by this run, remove it entirely
rm -rf "<TARGET_DIR>"

# If the target directory already existed, remove only what was added:
rm -f "<TARGET_DIR>/<SOLUTION_FILE>"         # .sln or .slnx
rm -rf "<TARGET_DIR>/<PROJECT_NAME>"
rm -f "<TARGET_DIR>/Directory.Build.props"   # if created
rm -f "<TARGET_DIR>/Directory.Packages.props" # if created
```

> Do **not** delete the `.git` folder or any files that existed before this skill ran.
> After cleanup, clearly explain what went wrong and what the user can do to fix it.

---

## Cross-Platform Notes

| Concern | Linux/macOS | Windows |
|---|---|---|
| Shell | bash | PowerShell |
| Path separator | `/` | `\` (but `dotnet` accepts `/`) |
| Filename case | Case-sensitive: `Directory.Build.props` must match exactly | Case-insensitive |
| Tree listing | `find … \| sort` | `Get-ChildItem -Recurse` |
| Create directory | `mkdir -p` | `New-Item -ItemType Directory` |
| Delete files | `rm -f` / `rm -rf` | `Remove-Item -Force` / `Remove-Item -Recurse -Force` |
| `dotnet` in PATH | Usually `/usr/bin/dotnet` | Usually `C:\Program Files\dotnet\dotnet.exe` |

The `dotnet` CLI itself behaves identically on all platforms — all `dotnet` commands in this skill work unchanged.

**Important:** Always use **absolute paths** in all commands. Do not rely on `cd` — the working directory may reset between terminal invocations.

---

## Available Templates

Do **not** use any hardcoded or predefined template list. Always run:

```bash
dotnet new list --type project
```

Show the user the **complete, unfiltered** CLI output (without a `--language` filter — the Language column lets the user see which templates support their chosen language). The user picks a template by its short name. Different machines may have different templates installed (e.g. Avalonia, Playwright, custom company templates) — the CLI output is the only source of truth.

---

## Example: Full Happy Path

```
Target dir:    /home/user/projects/my-solution   (empty)
Solution name: MyApp
Project name:  MyApp.Api
Template:      webapi
Language:      C#
Framework:     net9.0
Props:         Directory.Build.props = yes, Directory.Packages.props = no
```

Commands executed:
```bash
dotnet new sln --name MyApp --output /home/user/projects/my-solution
# Check: did it create MyApp.sln or MyApp.slnx? (SDK 9+ defaults to .slnx)
ls /home/user/projects/my-solution/MyApp.*   # → MyApp.slnx

dotnet new webapi --name MyApp.Api --output /home/user/projects/my-solution/MyApp.Api --language "C#" --framework net9.0
dotnet sln /home/user/projects/my-solution/MyApp.slnx add /home/user/projects/my-solution/MyApp.Api/MyApp.Api.csproj
# create Directory.Build.props
dotnet build /home/user/projects/my-solution -v minimal
```

Result:
```
my-solution/
├── MyApp.slnx
├── Directory.Build.props
└── MyApp.Api/
    ├── MyApp.Api.csproj
    ├── Program.cs
    └── ...
```
