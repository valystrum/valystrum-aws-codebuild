# Changelog

Each release is a tag of this repository. Copy the scripts of the release you want into `.valystrum/` (see the
README's Quick start); to update, copy them again from a newer release.

## v1.0.0 (2026-10-29)

The first release, for Valystrum v0.21.0 and later.

- `valystrum-scan.sh` with `examples/buildspec-security-scan.yml`: leaked secrets, code analysis (SAST) and IaC checks
  on a pull request's changed files, the findings as test cases in the build's Reports tab, and the build failed on a
  critical or high finding.
- `valystrum-sbom.sh` with `examples/buildspec-sbom.yml`: a CycloneDX SBOM made with the official CycloneDX generators
  (pinned, their dependencies locked by hash) and uploaded to your Valystrum software inventory, with
  `VALYSTRUM_COMPLETE` to remove what the SBOM no longer lists.
- The same buildspecs serve a CodePipeline (V2) build stage.
