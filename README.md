# Renovate discussion 47267

Minimal reproduction targeting a wildcard-generation problem observed with Renovate **44.175.2**, using native NuGet versioning and `rangeStrategy: "bump"`.

The repository contains one dependency: `Microsoft.NET.Test.Sdk` with the floating version `17.*` in `Reproduction.csproj`.  
It uses the public NuGet feed, with no credentials or private packages.

The checked-in `renovate.json` explicitly sets `rangeStrategy: "bump"` for NuGet packages.

## Current behavior

Starting from `Microsoft.NET.Test.Sdk` declared as `17.*`, Renovate 44.175.2 generated two updates in a real run. A subsequent full dry run confirmed the same calculated declarations:

| Current value | Target release | Observed new value | Observed PR action                  |
|---------------|----------------|--------------------|-------------------------------------|
| `17.*`        | `18.11.0`      | `18.1*`            | Created a major-update PR.          |
| `17.*`        | `17.14.1`      | `17.1*`            | Created a separate minor-update PR. |

The major update **is detected**. The problem is that its generated declaration is `18.1*` instead of `18.*`.

The same wildcard-generation problem also changes `17.*` to `17.1*` for the 17.x target. Preserving the existing whole-component wildcard would instead leave `17.*` unchanged, so no package-file update or separate PR would be needed for that target.

These observations come from a real run and a subsequent dry run in an existing repository, not a completed run against this newly prepared repository. Results may differ with newer Renovate versions or future NuGet feed contents.

### Reproduction steps

1. Run Renovate (44.175.2) against this repository with debug logging and the checked-in configuration.
2. Inspect the lookup result for `Microsoft.NET.Test.Sdk`, confirming native `nuget` versioning and the `bump` range strategy.
3. Inspect `newValue` for an update targeting `18.11.0`. The observed incorrect value was `18.1*`; the expected value is `18.*`.
4. Inspect the separate update targeting `17.14.1`. The observed value was `17.1*`;  
   the expected outcome is to retain `17.*` and skip this package-file update. In a real run, this should not create a separate PR for this dependency.

## Expected behavior

When `bump` generates a new floating declaration, it should preserve the position and form of the complete dotted wildcard:

| Current value | Target release | Expected declaration | Expected PR behavior                                           |
|---------------|----------------|----------------------|----------------------------------------------------------------|
| `17.*`        | `18.11.0`      | `18.*`               | Propose the major update with the complete wildcard preserved. |
| `17.*`        | `17.14.1`      | `17.*` (unchanged)   | Skip the package-file update; no separate PR is needed.        |

The requested behavior is to preserve the existing whole-component wildcard, not to add digits from the target version to it. For a target already covered by the floating pattern, such as `17.14.1` under `17.*`, that produces an unchanged declaration and therefore no manifest update.

The defect is the replacement of a complete wildcard with a partial wildcard, not the detection of the major update.

### NuGet version handling

NuGet represents `17.*` as an open-ended minimum of `17.0.0` plus a floating preference for the highest matching 17.x release. Thus, `18.11.0` satisfies the general version bounds but does not match the 17.x floating pattern.

Renovate's `bump` strategy allows updates even when the target satisfies the existing version bounds. For this reproduction, the expected declaration preserves the complete wildcard: `18.*` for the major update and unchanged
`17.*` for a 17.x target.  
[Renovate range strategies](https://docs.renovatebot.com/configuration-options/#rangestrategy).

## Link to the Renovate issue or Discussion

https://github.com/renovatebot/renovate/discussions/47267
