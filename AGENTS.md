# Agent instructions

Workspace conventions load globally via `~/.claude/CLAUDE.md` -> `agentic-os-kai/AGENTS.md`. This file covers only what is specific to this repo.

## Scope

EcoTelemetry is an OpenTelemetry observability mod for Eco game servers. Public repo on GitHub. v1 captures exceptions via OTLP logs; metrics and traces follow.

## Project shape

Eco loads precompiled mods from `Mods/<ModName>/` at server startup. EcoTelemetry ships as a single ZIP containing:

- `EcoTelemetry.dll`
- All transitive OpenTelemetry NuGet DLLs (the build copies them to `bin/Release/net10.0/`)
- `EcoTelemetry.example.json` (copied to the server's `Configs/` folder by the operator)

Design notes behind the source live in [docs/internals.md](docs/internals.md).

## Repo boundaries

This repo is **public**. Do not cite filenames or line numbers from `StrangeLoopGames/Eco` (proprietary game source) in any committed file. Knowledge from the source is fine. Pointers into it are not. Anchor references in:

- <https://wiki.play.eco/en/Modding>
- <https://docs.play.eco/>
- <https://github.com/StrangeLoopGames/EcoModKit>

Same rule applies to private sibling repos under `coilysiren/` (`eco-mods`, `eco-configs`, `eco-mods-assets`, `eco-mods-assets-embeded`).

## Commands

Route every dev command through ward, which reads [`.ward/ward.yaml`](.ward/ward.yaml) (run verbs with `ward exec <verb>`). The lockdown denies bare invocations of the underlying tools (`dotnet`, etc.). Add new verbs to that file before invoking them.

## Validation

```bash
ward exec build
```

The csproj targets `net10.0` (matches `EcoServerTargetFramework` in current Eco) and pulls `Eco.ReferenceAssemblies` from NuGet for type-check. Real validation only happens when the DLL is deployed onto a running server.

## Safety

Never commit secrets or opaque ids (route to AWS SSM per `../AGENTS.md`). Never leak proprietary Eco source. Do not auto-trigger an Eco server reload; flag a pending reload instead.

## Cross-repo contracts

Depends on the `Eco.ReferenceAssemblies` NuGet package, pinned to the current Eco `EcoServerTargetFramework`. No forward or backward compatibility guarantees across Eco versions.

## Release

The release workflow in `.github/workflows/release.yml` builds the assembly and produces the distribution ZIP. CI then ssh-installs into `/home/kai/Steam/steamapps/common/EcoServer/Mods/UserCode/EcoTelemetry/` (no sudo, NoSudo path). The running Eco server picks up the mod on next reload.

## Agent rules

After pushing to `main`, schedule a wake-up (~360s) to verify build + release + on-host install landed on kai-server:

- **Verify CI**: `coily gh run list --repo coilysiren/eco-telemetry --limit 1` should show `completed/success`. Re-schedule once at +180s if in progress; stop on failure.
- **Verify install**: `coily ssh kai-server ls .../Mods/UserCode/EcoTelemetry/` should show the freshly written assembly. Mismatched mtime versus push time means the deploy didn't run.
- **Reload**: the mod takes effect after the server reloads (restart or in-game `/reload`). Don't trigger it; note a pending reload in the wake-up summary.

## See also

- [README.md](README.md) - human-facing intro.
- [docs/FEATURES.md](docs/FEATURES.md) - inventory of what ships today.
- [.coily/coily.yaml](.coily/coily.yaml) - allowlisted commands.
- [.ward/ward.yaml](.ward/ward.yaml) - allowlisted commands (`ward exec`).

Cross-reference convention from [coilysiren/agentic-os#59](https://github.com/coilyco-flight-deck/agentic-os/issues/59).
