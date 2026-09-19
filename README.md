# Component Detection integration fixtures

This public repository is a durable end-to-end test for
[`advanced-security/component-detection-dependency-submission-action`](https://github.com/advanced-security/component-detection-dependency-submission-action).
Its dependencies are intentionally small, real, and pinned. Keep them patched and avoid adding packages solely to increase ecosystem coverage.

## Fixtures

| Ecosystem | Fixture | Component Detection detector/input | Restore command | Expected repository-relative manifest path |
| --- | --- | --- | --- | --- |
| npm | Root application using `is-number` | Stable npm lockfile detector, `package-lock.json` | `npm ci` | `package-lock.json` |
| npm | Nested frontend using `picocolors` | Stable npm lockfile detector, `package-lock.json` | `npm ci --prefix frontend` | `frontend/package-lock.json` |
| NuGet | SDK-style .NET library using `Newtonsoft.Json` | Stable `NuGetProjectCentric`/`MSBuildBinaryLog` path, generated `obj/project.assets.json` | `dotnet restore dotnet/ComponentDetectionTest.csproj` | `dotnet/obj/project.assets.json` |
| Maven | Java project using `commons-codec` | Stable Maven CLI detector, `pom.xml` | `mvn --batch-mode --file maven/pom.xml dependency:go-offline` | `maven/pom.xml` |
| Go | Module using `github.com/google/uuid` | Stable Go detector, `go.mod` and `go.sum` | `go mod download` from `go/` | `go/go.mod` |
| Ruby | Bundler project using `rake` | Stable RubyGems detector, `Gemfile.lock` | `bundle install --gemfile ruby/Gemfile` | `ruby/Gemfile.lock` |
| Python | Pinned `certifi` requirement | Stable pip report detector, `requirements.txt` plus generated pip installation report | `python -m pip install --report python/component-detection-pip-report.json -r python/requirements.txt` | `python/requirements.txt` |

The workflow restores every fixture before running Component Detection. NuGet relies on the generated
`dotnet/obj/project.assets.json`; `packages.lock.json` is not used or claimed as detector input. The Python report is generated beside
`requirements.txt` during the workflow and is intentionally ignored by Git. Maven and other ecosystem tools may resolve transitive
dependencies, so the submitted graph can contain more components than the direct dependencies listed above.

The dependency submission uses the stable `multi-ecosystem-integration` correlator so repeated runs replace the same logical snapshot.
