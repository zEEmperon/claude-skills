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

Ask the user for **all of the following** before proceeding. You can ask in one message:

1. **Target directory** — full path where the solution will be created. Must exist and be empty (a `.git` folder is allowed).
2. **Solution name** — e.g. `MyApp`
3. **Project name** — e.g. `MyApp.Api`
4. **Language** — `C#` (default), `F#`, or `VB`. Note: not all templates are available for every language.
5. **Project template** — run `dotnet new list --type project --language "<CHOSEN_LANGUAGE>"` first and show the user the results so they can choose. Let the user pick by short name.
6. **Target framework** (optional) — e.g. `net9.0`, `net8.0`. Leave blank to use the SDK default.
7. **Directory.Build.props** — yes/no. Centralizes shared MSBuild properties (TreatWarningsAsErrors, Nullable, etc.) across all projects in the solution.
8. **Directory.Packages.props** (Central Package Management) — yes/no. Only relevant if the user plans multiple projects or wants to centralize NuGet versions. Requires NuGet 6.2+ / .NET SDK 6.0.300+.

> **Never skip asking for the target directory.** Refuse to proceed without it.

---

## Step 2 — Pre-flight Checks

Before running any `dotnet` commands, validate everything via `bash_tool`:

```bash
# 1. dotnet SDK present
dotnet --version

# 2. Directory exists
test -d "<TARGET_DIR>" && echo "exists" || echo "missing"

# 3. Directory is empty (allow only .git)
find "<TARGET_DIR>" -mindepth 1 -maxdepth 1 ! -name '.git' | head -5
```

**Rules:**
- If `dotnet` is missing → tell the user to install the .NET SDK from https://dotnet.microsoft.com/download and stop.
- If directory is missing → create it automatically with `mkdir -p "<TARGET_DIR>"` and inform the user it was created.
- If directory has unexpected files → stop and warn; do **not** overwrite.

---

## Step 3 — Scaffold the Solution

```bash
cd "<TARGET_DIR>"
dotnet new sln --name "<SOLUTION_NAME>" --output .
```

> `--output .` keeps the `.sln` file in the target directory (not a subdirectory).

---

## Step 4 — Scaffold the Project

```bash
# Basic (no framework specified)
dotnet new <TEMPLATE> --name "<PROJECT_NAME>" --output "<PROJECT_NAME>" --language "<LANGUAGE>"

# With explicit framework
dotnet new <TEMPLATE> --name "<PROJECT_NAME>" --output "<PROJECT_NAME>" --language "<LANGUAGE>" --framework <FRAMEWORK>
```

The project lands in `<TARGET_DIR>/<PROJECT_NAME>/`.

---

## Step 5 — Add Project to Solution

```bash
dotnet sln "<TARGET_DIR>/<SOLUTION_NAME>.sln" add "<TARGET_DIR>/<PROJECT_NAME>/<PROJECT_NAME>.csproj"
```

---

## Step 6 — Optional: Directory.Build.props

If the user chose yes, create `<TARGET_DIR>/Directory.Build.props`. Tailor the content to the chosen language:

**C# (full set of properties):**
```xml
<Project>
  <PropertyGroup>
    <!-- Enforce nullable reference types across all projects -->
    <Nullable>enable</Nullable>
    <!-- Treat all warnings as errors in CI; comment out locally if noisy -->
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
    <!-- Implicit usings reduce boilerplate -->
    <ImplicitUsings>enable</ImplicitUsings>
    <!-- Deterministic builds for reproducibility -->
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

**F# or VB (skip C#-only properties):**
```xml
<Project>
  <PropertyGroup>
    <!-- Treat all warnings as errors in CI; comment out locally if noisy -->
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
    <!-- Deterministic builds for reproducibility -->
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
    <!-- Enable NuGet Central Package Management (requires NuGet 6.2+ / SDK 6.0.300+) -->
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!--
      Define package versions here. Projects reference packages WITHOUT a Version attribute.
      Example:
      <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
      <PackageVersion Include="Serilog" Version="4.1.0" />
    -->
  </ItemGroup>
</Project>
```

When `Directory.Packages.props` is created, also strip the `Version` attribute from any `<PackageReference>` items generated by the project template (if any). Use `str_replace` or `sed` to patch the `.csproj`.

---

## Step 8 — Verify the Result

```bash
# Show final tree (Linux/macOS)
find "<TARGET_DIR>" -not -path '*/.git/*' | sort

# Build to confirm everything compiles
cd "<TARGET_DIR>"
dotnet build --no-restore -v minimal 2>&1 | tail -20
```

On Windows use `Get-ChildItem -Recurse "<TARGET_DIR>"` instead of `find`.

---

## Step 9 — Present the Result

Report:
- ✅ Files created (solution, project, optional props files)
- 📁 Directory tree
- 🔨 Build status (pass/fail)
- Next steps the user can take (e.g., `dotnet run`, adding packages with `dotnet add package <Name>`)

---

## Cleanup on Failure

If **any** step fails, roll back everything created in this run. Track which artifacts were created:

```bash
# If the target directory itself was created by this run (did not exist before), remove it entirely
rm -rf "<TARGET_DIR>"

# If the target directory already existed before this run, remove only what was added:
rm -f "<TARGET_DIR>/<SOLUTION_NAME>.sln"
rm -rf "<TARGET_DIR>/<PROJECT_NAME>"
rm -f "<TARGET_DIR>/Directory.Build.props"
rm -f "<TARGET_DIR>/Directory.Packages.props"
```

> Do **not** delete the `.git` folder or any files that existed before this skill ran.
> After cleanup, clearly explain what went wrong and what the user can do to fix it.

---

## Cross-Platform Notes

| Concern | Linux/macOS | Windows |
|---|---|---|
| Path separator | `/` | `\` (but `dotnet` accepts `/`) |
| Filename case | Case-sensitive: `Directory.Build.props` must match exactly | Case-insensitive |
| Tree listing | `find "<TARGET_DIR>" -not -path '*/.git/*' \| sort` | `Get-ChildItem -Recurse "<TARGET_DIR>"` |
| `dotnet` in PATH | Usually `/usr/bin/dotnet` | Usually `C:\Program Files\dotnet\dotnet.exe` |

The `dotnet` CLI itself behaves identically on all platforms — all `dotnet` commands in this skill work unchanged.

---

## Common dotnet new Templates by Language

Always run `dotnet new list --type project --language "<CHOSEN_LANGUAGE>"` to show the exact templates available — the table below is a reference, not exhaustive.

**C#** (widest template support):

| Short name | Description |
|---|---|
| `console` | Console Application |
| `classlib` | Class Library |
| `webapi` | ASP.NET Core Web API |
| `web` | ASP.NET Core Empty |
| `mvc` | ASP.NET Core MVC |
| `razor` | ASP.NET Core Razor Pages |
| `blazorserver` | Blazor Server App |
| `blazorwasm` | Blazor WebAssembly App |
| `worker` | Worker Service |
| `grpc` | ASP.NET Core gRPC Service |
| `mstest` | MSTest Unit Test Project |
| `xunit` | xUnit Test Project |
| `nunit` | NUnit Test Project |

**F#** (functional-first subset):

| Short name | Description |
|---|---|
| `console` | Console Application |
| `classlib` | Class Library |
| `webapi` | ASP.NET Core Web API |
| `web` | ASP.NET Core Empty |
| `worker` | Worker Service |
| `mstest` | MSTest Unit Test Project |
| `xunit` | xUnit Test Project |
| `nunit` | NUnit Test Project |

**VB (Visual Basic)**:

| Short name | Description |
|---|---|
| `console` | Console Application |
| `classlib` | Class Library |
| `winforms` | Windows Forms App |
| `wpf` | WPF Application |
| `mstest` | MSTest Unit Test Project |

---

## Example: Full Happy Path

```
Target dir:    /home/user/projects/my-solution   (empty)
Solution name: MyApp
Project name:  MyApp.Api
Template:      webapi
Framework:     net9.0
Props:         Directory.Build.props = yes, Directory.Packages.props = no
```

Commands executed:
```bash
cd /home/user/projects/my-solution
dotnet new sln --name MyApp --output .
dotnet new webapi --name MyApp.Api --output MyApp.Api --language "C#" --framework net9.0
dotnet sln MyApp.sln add MyApp.Api/MyApp.Api.csproj
# create Directory.Build.props
dotnet build --no-restore -v minimal
```

Result:
```
my-solution/
├── MyApp.sln
├── Directory.Build.props
└── MyApp.Api/
    ├── MyApp.Api.csproj
    ├── Program.cs
    └── ...
```
