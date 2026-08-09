# TrueNAS CSI driver

This directory installs the official [TrueNAS CSI driver](https://github.com/truenas/truenas-csi) using a vendored upstream deployment manifest and local Kustomize patches.

## Why the upstream manifest is vendored

TrueNAS CSI does not currently provide a supported Helm chart or OCI-compatible installation artifact for deploying the driver. Until one is available, the upstream deployment manifest is copied into this repository from a pinned release.

The vendored manifest under `upstream/` is kept unchanged. Environment-specific configuration and required compatibility changes are maintained independently under `patches/` and applied by the local `kustomization.yaml`.

This separation provides:

- a recognizable copy of the upstream deployment manifest;
- reviewable local changes that are not mixed into upstream YAML;
- simpler upgrades by replacing the base manifest and then validating the existing patches;
- a clear distinction between upstream behavior and homelab-specific configuration;
- immutable image selection through a pinned digest.

Local patches remove the placeholder API-key Secret, configure the TrueNAS connection, apply control-plane scheduling and namespace security settings, and provide Talos-specific compatibility where required.

## Upstream source

- Repository: <https://github.com/truenas/truenas-csi>
- Release: `v1.1.2`
- Commit: `d5fddc049d1cc80da9bb1f37569a8b49a7956dce`
- Source file: `deploy/truenas-csi-driver.yaml`
- Driver image: `ghcr.io/truenas/truenas-csi:v1.1.2`
- Pinned OCI index digest: `sha256:72bc60182c3eb22b524f52c7693ffb2fcd8e8f75ac47c0a2ce0239e78af6bf83`

## Updating the vendored release

When upgrading TrueNAS CSI:

1. Select and record an immutable upstream release and commit.
2. Replace `upstream/truenas-csi-driver.yaml` with the manifest from that release without adding local configuration to it.
3. Update the driver image digest in `kustomization.yaml`.
4. Reapply and review every local patch against the new base.
5. Render the complete Kustomize output and validate the CSI deployment and volume lifecycle before promotion.
6. Update the provenance information in this README.

## Future migration

When TrueNAS provides a supported Helm chart or OCI-compatible installation method, migrate away from the vendored-manifest approach. Prefer a pinned OCI source and Helm release while preserving the same configuration, secret-management, scheduling, security, and compatibility requirements through chart values or narrowly scoped post-render patches where necessary.
