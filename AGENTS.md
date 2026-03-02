# AGENTS.md

This is the fast navigation layer for this repo. Use it to find the right part of the full RealityScan / RealityCapture CLI guide quickly.

## How To Use This Repo

1. Start here in `AGENTS.md` if you need to route yourself to the right topic quickly.
2. Open `RealityScan_CLI_combined_docs_for_agents.md` for the actual detailed instructions and full command reference.
3. Use `README.md` as the public repo landing page, not as the main technical reference.

## File Roles

- `README.md`: short public summary of what the repo contains.
- `AGENTS.md`: fast navigation map for agents.
- `RealityScan_CLI_combined_docs_for_agents.md`: the full source of truth.

## Fast Task Routing

| If you need... | Go to this section | Likely commands / artifacts |
| --- | --- | --- |
| start from scratch with images | `Inputs and Project Setup` | `-newScene`, `-add`, `-addFolder`, `-save` |
| import video frames | `Inputs and Project Setup` | `-importVideo` |
| import LiDAR / scans | `Inputs and Project Setup` | `-importLaserScan`, `-importLaserScanFolder`, `params.xml` |
| align inputs | `Alignment and Component Control` | `-detectFeatures`, `-align`, `-draft` |
| choose the right component | `Alignment and Component Control` | `-selectMaximalComponent`, `-selectComponent`, `-selectComponentWithLeastReprojectionError` |
| reuse alignment in another project | `Alignment and Component Control` | `-exportXMP`, `-exportRegistration`, component export/import commands |
| define or reuse reconstruction bounds | `Reconstruction Region and Model Calculation` | `.rsbox`, `-setReconstructionRegionAuto`, `-importReconstructionRegion`, `-exportReconstructionRegion` |
| compute preview / normal / high models | `Reconstruction Region and Model Calculation` | `-calculatePreviewModel`, `-calculateNormalModel`, `-calculateHighModel`, `-continueModelCalculation` |
| texture / simplify / clean a model | `Model Processing and Deliverables` | simplify, smoothing, texturing, cleanup commands |
| export a model | `Model Processing and Deliverables` | `-exportModel`, `-exportSelectedModel`, `-exportModelToZip`, `params.xml` |
| generate ortho outputs | `Model Processing and Deliverables` and `Settings, Presets, and External Parameter Files` | `.rcortho`, `-calculateOrthoProjection` |
| set runtime behavior and import/export config files | `Settings, Presets, and External Parameter Files` | `-set`, `params.xml`, `.rsbox`, `.rcortho` |
| run headless and monitor progress | `Reliability, Headless Runs, and Monitoring` | `-headless`, `-hideUI`, `-silent`, `-writeProgress`, `-printProgress` |
| use multiple instances | `Delegation and Multi-Instance Control` | `-setInstanceName`, `-delegateTo`, `-waitCompleted`, `-getStatus` |
| use `.rscmd` / `.rccmd` | `Command Files (.rscmd / .rccmd)` | `-execRSCMD`, `-execRSCMDIndirect` |
| look up a specific command name quickly | `Complete CLI Command Reference` | search the appendix directly |

## Section Map For The Full Guide

These are the exact top-level sections in `RealityScan_CLI_combined_docs_for_agents.md`:

- `What This File Covers`
- `Core Operating Model`
- `State Dependencies`
- `Task-to-Command Map`
- `Standard Workflow Lifecycle`
- `Inputs and Project Setup`
- `Alignment and Component Control`
- `Reconstruction Region and Model Calculation`
- `Model Processing and Deliverables`
- `Settings, Presets, and External Parameter Files`
- `Reliability, Headless Runs, and Monitoring`
- `Delegation and Multi-Instance Control`
- `Command Files (.rscmd / .rccmd)`
- `Non-CLI Source Pages (Brief Notes)`
- `Complete CLI Command Reference`
- `Source Coverage and Verification Notes`

Rule of thumb:

- If you need the operating logic, use the upper workflow sections.
- If you need exact syntax, jump to `Complete CLI Command Reference`.

## Artifact-Based Routing

- `params.xml`: import/export dialog settings; see `Settings, Presets, and External Parameter Files` and the relevant command entry in `Complete CLI Command Reference`.
- `.rsbox`: reconstruction region definition; see `Reconstruction Region and Model Calculation`.
- `.rcortho`: orthographic projection parameters; see `Settings, Presets, and External Parameter Files` and ortho-related command entries.
- `.rscmd`: command batch file; see `Command Files (.rscmd / .rccmd)`.
- `.rccmd`: same batching role as `.rscmd`; see `Command Files (.rscmd / .rccmd)`.

## Command-Family Routing

If you already know the command family, jump into the matching appendix category under `Complete CLI Command Reference`:

- `Project and Images`
- `Delegate commands`
- `Alignment`
- `Reconstruction`
- `Model Tools`
- `Settings' and Error-handling Commands`

## Legacy and Alias Notes

The source HTML includes a few example-era or non-standard tokens that can show up in older snippets:

- `-exportComponent`
- `-minComponentSize`
- `-setReconRegionOnCPs`
- RSNode flags: `-hostAddress`, `-port`, `-landingPage`

These appear in examples or the Node page. The canonical command forms are documented in the full guide, and the main command reference is based on the master command table.

## Quick Rule

If you know the task, use the workflow sections. If you know the command family, use the appendix categories. If you know the exact command name, search the full guide directly.
