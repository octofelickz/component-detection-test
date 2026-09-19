# Component Detection integration fixtures

This public repository is a durable end-to-end test for
[`advanced-security/component-detection-dependency-submission-action`](https://github.com/advanced-security/component-detection-dependency-submission-action).
Its dependencies are intentionally small, real, and pinned. Keep them patched and avoid adding packages solely to increase ecosystem coverage.

## Fixtures

| Ecosystem | Fixture | Component Detection detector/input | Restore command | Expected repository-relative manifest path |
| --- | --- | --- | --- | --- |
| npm | Root application using `is-number` | Stable npm lockfile detector, `package-lock.json` | `npm ci` | `package-lock.json` |
| npm | Nested frontend using `picocolors` | Stable npm lockfile detector, `package-lock.json` | `npm ci --prefix frontend` | `frontend/package-lock.json` |
| NuGet | SDK-style .NET library using direct `Serilog.Sinks.Console` 6.0.0 and transitive `Serilog` 4.0.0 | Stable `NuGetProjectCentric`/`MSBuildBinaryLog` path, generated `obj/project.assets.json` | `dotnet restore dotnet/ComponentDetectionTest.csproj` | `dotnet/ComponentDetectionTest.csproj` |
| Maven | Java project using `commons-codec` | Stable Maven CLI detector, `pom.xml` | `mvn --batch-mode --file maven/pom.xml dependency:go-offline` | `maven/pom.xml` |
| Go | Module using `github.com/google/uuid` | Stable Go detector, `go.mod` and `go.sum` | `go mod download` from `go/` | `go/go.mod` |
| Ruby | Bundler project using `rake` | Stable RubyGems detector, `Gemfile.lock` | `bundle install --gemfile ruby/Gemfile` | `ruby/Gemfile.lock` |
| Python | Pinned `certifi` requirement | Stable pip report detector, `requirements.txt` plus generated pip installation report | `python -m pip install --report python/component-detection-pip-report.json -r python/requirements.txt` | `python/requirements.txt` |

The workflow restores every fixture before running Component Detection. NuGet relies on the generated
`dotnet/obj/project.assets.json`; `packages.lock.json` is not used or claimed as detector input. The submitted NuGet manifest should show
`Serilog.Sinks.Console` as direct and `Serilog` as transitive, both sourced from `dotnet/ComponentDetectionTest.csproj`. The Python report is generated beside
`requirements.txt` during the workflow and is intentionally ignored by Git. Maven and other ecosystem tools may resolve transitive
dependencies, so the submitted graph can contain more components than the direct dependencies listed above.

The dependency submission uses the stable `multi-ecosystem-integration` correlator so repeated runs replace the same logical snapshot.

## Expected .NET SDK behavior

Component Detection may retain an internal graph node such as `10.0.400 net8.0 unknown - DotNet`. That observed
detector behavior is not an actionable package dependency: the node must be omitted from `componentsFound` and from
the submitted dependency snapshot. SDK, runtime, and framework components are serviced through SDK, runtime, operating
system, or base-image updates rather than through a project `PackageReference`, so the SDK node must not be persisted in
the repository dependency graph.

This exclusion is distinct from the framework conflict-resolution rationale documented for
[`NuGetProjectCentric`](https://github.com/microsoft/component-detection/blob/main/docs/detectors/nuget.md#nugetprojectcentric).
The current default-on `MSBuildBinaryLog` detector replaces the former `NuGetProjectCentric` and `DotNet` detectors.
During a build, the .NET SDK resolves conflicts by ignoring package assets that overlap newer assets supplied by the
target framework. Because that result is not persisted in a build artifact, Component Detection approximates it with
framework-specific package lists and marks losing packages as development dependencies. The documented
`PrunePackageReference` work moves similar conflict resolution into NuGet restore, where pruned packages are not
downloaded and therefore do not appear in `project.assets.json`. Those rules explain how conflicting
`PackageReference` packages are classified or removed; they do not justify submitting the SDK itself as a package
dependency.

The actionable expected .NET graph is:

| Component | Expected relationship | Source manifest |
| --- | --- | --- |
| `Serilog.Sinks.Console@6.0.0` | Direct | `dotnet/ComponentDetectionTest.csproj` |
| `Serilog@4.0.0` | Transitive dependency of `Serilog.Sinks.Console@6.0.0` | `dotnet/ComponentDetectionTest.csproj` |

The exported dependency graph SBOM exposes exact package identities and the
`Serilog.Sinks.Console@6.0.0 DEPENDS_ON Serilog@4.0.0` SPDX relationship. It does not currently expose the dependency
submission manifest/source metadata or direct/transitive labels, so the workflow verifies the supported SBOM evidence
without making fragile assertions about unavailable fields.
