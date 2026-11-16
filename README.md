# zeepkist-mod-workflows

Shared GitHub Actions workflows for releasing Zeepkist mods.

## Releasing a mod

Go to **Actions → release → Run workflow**, then pick:

- **bump**: `patch`, `minor` or `major`, applied to the latest stable tag
- **stage**: `stable`, or `alpha`/`beta`/`rc` for a pre-release

| Latest tags | bump + stage | Result |
|---|---|---|
| `1.0.3` | patch + stable | `1.0.4` → GitHub release + mod.io |
| `1.0.3` | minor + beta | `1.1.0-beta.1` → GitHub pre-release only |
| `1.0.3`, `1.1.0-beta.1` | minor + beta | `1.1.0-beta.2` |
| `1.0.3`, `1.1.0-beta.2` | minor + stable | `1.1.0` → GitHub release + mod.io |

Stable releases must run from the default branch. Pre-releases can run from any branch, which you pick under *Use workflow from*.

What happens:

1. **version**: reads the git tags and works out the next version. It fails early if the tag already exists, if there is no stable tag yet, or if a stable release is missing its mod.io id or secret.
2. **build** (Windows): runs the tests, then publishes the Debug build with the version applied (`-p:Version`). It runs the verify script if there is one, converts symbols to `.mdb` with a pinned and cached Mono, and zips the result.
3. **publish**: writes the version into the csproj and commits `chore: release <version>` on top of the built commit. A stable release pushes that commit to the default branch together with the tag, in one atomic push. A pre-release only pushes the tag, so the branch it ran from is unchanged. Then it creates a GitHub release with notes generated since the previous tag and the zip attached.
4. **modio** (stable only): uploads the zip to mod.io with the release notes as changelog.

Git tags decide the next version, and each release writes it back into the csproj. After a stable release, `<Version>` on the default branch matches the version on mod.io. Don't edit `<Version>` by hand; `version-check` rejects PRs that do.

Stable releases use `X.Y.Z` as the mod loader version. Pre-releases get a fourth number the mod loader understands: alpha → `.1xx`, beta → `.2xx`, rc → `.3xx`, where `xx` is the iteration. For example, `1.1.0-beta.2` becomes `1.1.0.202`, and the tag's csproj says so too.

Stable releases push to the default branch with the workflow's `GITHUB_TOKEN`. If a branch ruleset requires status checks, add **GitHub Actions** as a bypass actor, or the push is rejected.

## Adding a mod

1. Copy [`templates/release.yml`](templates/release.yml) and [`templates/version-check.yml`](templates/version-check.yml) into the mod's `.github/workflows/`.
2. Set `mod-id`, and any optional inputs the mod needs.
3. Make sure a starting tag exists for the version currently on mod.io, e.g. `git tag 1.2.3 <commit> && git push origin 1.2.3`.
4. `MODIO_BEARER` comes from the `donderjoekel` org secrets through `secrets: inherit`.

### Inputs

| Input | Default | Purpose |
|---|---|---|
| `bump` | required | `patch`, `minor` or `major` |
| `stage` | `stable` | `stable`, `alpha`, `beta` or `rc` |
| `mod-id` | `0` | mod.io mod id (required for stable) |
| `project` | the only root `.csproj` | Path to the mod's csproj |
| `test-projects` | none | Test projects to run, one per line |
| `pdb2mdb-extra` | none | Extra DLLs in the output to convert, one per line |
| `pre-build-script` | none | PowerShell script run after restore |
| `verify-script` | none | PowerShell script run with the publish directory as argument |
| `dotnet-version` | `8.0.x` | SDK version(s) for setup-dotnet |

## Versioning this repo

Mods reference `@v1`. Backwards-compatible changes move the `v1` tag. Breaking changes get a `v2`.
