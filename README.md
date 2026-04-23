# paket-mixed-nuget-paket

## Probe metadata

- **Pattern**: mixed-nuget-paket
- **Target framework**: net8.0
- **Storage mode**: none
- **Generated**: 2026-04-22
- **Purpose**: Validates that Mend correctly handles a hybrid solution where Paket and standard NuGet PackageReference coexist. Both package managers must be detected independently, producing separate project entries with correct dependency attribution.

## Feature exercised

This probe exercises the coexistence of two distinct .NET dependency management mechanisms in one solution:

1. `PaketProject` — a Paket-managed library. It has a `paket.references` file listing its packages (`Newtonsoft.Json`, `Serilog`) and a `.csproj` that imports `paket.targets` but contains no `<PackageReference>` elements. Its dependencies are resolved from the shared `paket.lock`.
2. `NuGetProject` — a standard NuGet-managed library. It has no `paket.references` file and no `paket.targets` import. Its packages (`Microsoft.Extensions.Logging 8.0.0`, `AutoMapper 12.0.1`) are declared directly via `<PackageReference>` elements in the `.csproj`. There is no `packages.lock.json` — Mend must use resolver-driven detection against the NuGet registry.

This pattern is common in real codebases that are incrementally migrating from NuGet to Paket (or the reverse). It tests whether Mend's scanner correctly identifies and applies separate detection strategies for each project within the same solution.

## Project layout

```
MixedSolution.sln
paket.dependencies          <- Paket constraints for PaketProject only
paket.lock                  <- Paket lockfile for PaketProject only
src/
  PaketProject/
    PaketProject.csproj     <- imports paket.targets; no PackageReference
    paket.references        <- Newtonsoft.Json, Serilog
  NuGetProject/
    NuGetProject.csproj     <- PackageReference only; no paket.references
```

## Expected dependency tree

Mend must produce 2 project entries in the dependency tree — one per `.csproj`. Each project is detected by a different mechanism.

### PaketProject (detected via Paket / lockfile-driven)

Source: `paket.lock` (root-level), scoped by `src/PaketProject/paket.references`.

Direct dependencies:
- `Newtonsoft.Json` 13.0.3 — no transitive dependencies
- `Serilog` 3.1.1 — no transitive dependencies

`dependencyFile` for all entries: `paket.lock`
`dependencyType` for all entries: `NUGET`

### NuGetProject (detected via NuGet PackageReference / resolver-driven)

Source: `src/NuGetProject/NuGetProject.csproj` PackageReference elements, resolved against nuget.org.

Direct dependencies:
- `Microsoft.Extensions.Logging` 8.0.0 — transitive children:
  - `Microsoft.Extensions.DependencyInjection.Abstractions` (>= 8.0.0, resolved 8.0.1)
  - `Microsoft.Extensions.Logging.Abstractions` (>= 8.0.0, resolved 8.0.0)
  - `Microsoft.Extensions.Options` (>= 8.0.0, resolved 8.0.0)
- `AutoMapper` 12.0.1 — transitive children:
  - `Microsoft.Extensions.DependencyInjection.Abstractions` (>= 6.0.0, resolved 8.0.1)

`dependencyFile` for all entries: `NuGetProject.csproj`
`dependencyType` for all entries: `NUGET`

## Key detection assertions

1. Mend must produce exactly **2 separate project entries**, one for `PaketProject` and one for `NuGetProject`.
2. `PaketProject` must be detected via Paket (lockfile-driven). Its `dependencyFile` must reference `paket.lock`, not a `.csproj`.
3. `NuGetProject` must be detected via NuGet PackageReference (resolver-driven). Its `dependencyFile` must reference the `.csproj` or a NuGet assets file, not `paket.lock`.
4. `PaketProject` must list `Newtonsoft.Json` and `Serilog` as roots. It must NOT contain `Microsoft.Extensions.Logging` or `AutoMapper`.
5. `NuGetProject` must list `Microsoft.Extensions.Logging` and `AutoMapper` as roots. It must NOT contain `Newtonsoft.Json` or `Serilog`.
6. The presence of `paket.lock` at the repo root must not cause Mend to incorrectly attribute NuGet PackageReference packages to Paket.
7. The absence of `paket.references` in `NuGetProject` must signal to Mend that Paket does not manage that project.
