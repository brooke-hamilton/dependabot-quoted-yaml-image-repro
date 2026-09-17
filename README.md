# Dependabot quoted YAML image reproduction

This public repository is a minimal reproduction of [dependabot/dependabot-core#7527](https://github.com/dependabot/dependabot-core/issues/7527), based on the quoted BuildKit image value that caused a hosted Dependabot update failure in the Radius Helm chart.

## Purpose

The repository isolates a GitHub-native Dependabot `docker` ecosystem update for a complete container image reference stored as a quoted YAML scalar:

```yaml
buildkit:
  image: "moby/buildkit:v0.13.2-rootless"
```

The Helm deployment template consumes `buildkit.image`, so this is a real chart value rather than an unused fixture. Dependabot operates directly on `values.yaml`.

## Expected behavior

Dependabot should detect that `moby/buildkit:v0.13.2-rootless` is outdated and rewrite the complete quoted scalar to the selected newer version while preserving valid YAML.

## Actual known behavior

GitHub-hosted Dependabot detects the dependency and available update, but the file updater fails with:

```text
RuntimeError: Expected content to change!
```

The failure originates from `Dependabot::Shared::SharedFileUpdater#updated_yaml_content`. The original Radius occurrence is visible in [this failed hosted Dependabot job](https://github.com/brooke-hamilton/radius/actions/runs/35230627204/job/105233482924).

## Reproduction

1. Enable GitHub-native Dependabot version updates by pushing [`.github/dependabot.yml`](.github/dependabot.yml).
2. Keep the outdated image in [`values.yaml`](values.yaml) exactly as `image: "moby/buildkit:v0.13.2-rootless"`.
3. Wait for the scheduled `docker` ecosystem update job, or trigger a supported update check from the repository dependency graph UI if available.
4. Observe that Dependabot parses the image and finds an update, then fails before rewriting `values.yaml`.

## Why the YAML is valid

YAML permits quoted string scalars. Quoting the complete image reference is valid YAML and is common in Helm values files. Helm parses the value as the string `moby/buildkit:v0.13.2-rootless`, and the chart renders it into the Deployment container image field.

## Captured hosted failure

This section will be updated after the GitHub-native Dependabot run for this repository completes. It will contain a concise excerpt from the actual hosted failure log, its UTC timestamp, and the public run/job link.
