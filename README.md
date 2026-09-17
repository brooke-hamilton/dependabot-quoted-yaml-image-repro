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

The GitHub-native Dependabot job started on 2026-09-17 at 15:14:24 UTC and failed at 15:15:02 UTC. See the [failed run](https://github.com/brooke-hamilton/dependabot-quoted-yaml-image-repro/actions/runs/35238900439) and its [Dependabot job](https://github.com/brooke-hamilton/dependabot-quoted-yaml-image-repro/actions/runs/35238900439/job/105261938411).

The following is a concise excerpt from the actual hosted job log, not expected output:

```text
2026/09/17 15:14:59 INFO Checking if moby/buildkit v0.13.2-rootless needs updating
2026/09/17 15:15:01 INFO Latest version is v0.33.0-rootless
2026/09/17 15:15:01 INFO Updating moby/buildkit from v0.13.2-rootless to v0.33.0-rootless
2026/09/17 15:15:02 ERROR Error processing moby/buildkit (RuntimeError)
2026/09/17 15:15:02 ERROR Expected content to change!
2026/09/17 15:15:02 ERROR /home/dependabot/docker/lib/dependabot/shared/shared_file_updater.rb:146:in 'Dependabot::Shared::SharedFileUpdater#updated_yaml_content'
```

This demonstrates a parser/updater mismatch. The Docker dependency parser successfully reads the quoted scalar and identifies the current image and a newer tag. The updater's `update_image` matching expects the image reference immediately after `image:` and therefore does not match the opening quote in `image: "moby/buildkit:v0.13.2-rootless"`. No replacement occurs, and `updated_yaml_content` raises because the returned content is unchanged.
