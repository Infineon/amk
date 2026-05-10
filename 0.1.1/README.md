# AMK 0.1.1 — Client Installation Guide

AMK ("AI/ML Kubeflow") is a self-contained, single-binary installer that deploys a
local Kubeflow platform on top of [KIND](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker
or Kubernetes-in-Podman). All container images are sourced from the public
HuggingFace dataset [`infineon/amk`](https://huggingface.co/datasets/infineon/amk),
so installation works on machines with restricted access to public registries
(Docker Hub, gcr.io, quay.io, registry.k8s.io, nvcr.io …).

This document is the entry point for **end users** running the binaries shipped
in this release. For developer/build documentation see the source repository.

---

## What's new in 0.1.1

Highlights since 0.1.0:

- **Embedded version string.** `amk --version` now prints the actual release
  number (`0.1.1`) instead of `0.0.0`. The version is also surfaced to in-cluster
  components as the `AMK_VERSION` environment variable.
- **Panels validation suite.** New `--panels` flag for `--command status`
  validates every Kubeflow Central Dashboard left-panel route (Notebooks,
  TensorBoards, Volumes, Katib, Pipelines, MLflow, Argo CD, Argo Workflows,
  Kubernetes Dashboard) plus the backing services. No browser required — runs
  fully headless and is CI-friendly.
- **Notebook readiness wait.** `create` now waits for Notebook controller and
  default notebook images to become ready before returning, removing the old
  race where the dashboard came up before the Notebooks tab was usable.
- **Status port-forward timing fix.** `--command status` no longer reports a
  spurious "port-forward not running" right after `create` / `set` while the
  forward is still settling.
- **Elyra / Jupyter improvements.** Elyra image now bakes in
  `kfp-kubernetes==1.1.0` plus `add_toleration` / `add_node_selector` /
  `add_pod_label` / `add_pod_annotation` stubs needed for KFP 2.4 driver
  compatibility, so Elyra-authored pipelines schedule correctly out of the box.
- **HuggingFace download robustness.** Multiple fixes around archive sync
  (retries, partial-download recovery, podman-aware paths). Cached archives in
  `~/.amk/images/` are reused as before.
- **Katib MySQL pinned from HF.** `katib-mysql` is now pinned to
  `mysql:8.0.36` and pulled exclusively from the HuggingFace archive — no
  fallback to public Docker Hub during install.
- **No Docker Hub login required.** Removed the optional `docker login` step;
  the HF-only image policy is now strictly enforced and any missing image fails
  fast with the exact archive URL that could not be resolved.
- **Pipeline-only NGC integration.** Optional NVIDIA NGC image source is now
  scoped to Kubeflow Pipelines runtime images only (cluster-level NGC pulls
  removed). The default flow remains 100% HuggingFace.
- **Windows: per-installation port allocation.** Multiple Windows installs on
  the same host no longer collide on `localhost:9002` — ports are allocated
  starting from `amk-1` (`9002`, `9003`, …) so each cluster gets its own.
- **Windows: assorted recover/podman fixes.** Smoother behavior for
  `--command recover` when WSL2 / Podman machine state is degraded.

The command-line surface is otherwise backwards compatible with 0.1.0.

---

## 1. Pick the right binary

The release ships four cross-compiled binaries. Pick the one matching your
machine and ignore the others.

| File                    | OS      | CPU    | Container runtime | Size    |
| ----------------------- | ------- | ------ | ----------------- | ------- |
| `amk-linux-amd64`       | Linux   | x86_64 | Docker            | ~7.5 MB |
| `amk-linux-arm64`       | Linux   | ARM64  | Docker            | ~7.1 MB |
| `amk-windows-amd64.exe` | Windows | x86_64 | Podman            | ~7.6 MB |
| `amk-windows-arm64.exe` | Windows | ARM64  | Podman            | ~7.2 MB |

> **Supported combinations are exactly these four.** Windows + Docker (with or
> without WSL) is not supported — on Windows you must use Podman Desktop.

After downloading, rename the file to `amk` (Linux/macOS) or `amk.exe`
(Windows) and place it somewhere on your `PATH`, e.g.:

```bash
# Linux
chmod +x amk-linux-amd64
sudo mv amk-linux-amd64 /usr/local/bin/amk
```

```powershell
# Windows (PowerShell)
Move-Item .\amk-windows-amd64.exe "$env:USERPROFILE\bin\amk.exe"
$env:Path += ";$env:USERPROFILE\bin"
```

The rest of this document uses `amk` / `amk.exe` interchangeably.

Verify the install:

```bash
amk --version       # prints: 0.1.1
```

---

## 2. Host prerequisites

Before installing AMK, make sure the following tools are present on `PATH`. The
binary itself is self-contained, but it shells out to the standard Kubernetes
toolchain on the host.

### Linux

| Tool      | Why it's needed                                          |
| --------- | -------------------------------------------------------- |
| `bash`    | Embedded deploy script is bash                           |
| `docker`  | Container runtime for KIND nodes                         |
| `kind`    | Builds the local cluster                                 |
| `kubectl` | Cluster admin + port-forwarding                          |
| `python3` | HuggingFace image-archive downloader (`huggingface_hub`) |

### Windows

| Tool                 | Why it's needed                                            |
| -------------------- | ---------------------------------------------------------- |
| **Podman Desktop**   | Provides `podman.exe` and the WSL2 Podman machine          |
| `kind.exe`           | Builds the local cluster (Podman provider)                 |
| `kubectl.exe`        | Cluster admin + port-forwarding                            |
| **Git for Windows**  | Provides `bash.exe` used to run the embedded deploy script |
| `python.exe` (3.10+) | HuggingFace image-archive downloader                       |

For an OS-aware, fully detailed checklist with install commands, run:

```bash
amk --requirements        # Linux
amk.exe --requirements    # Windows
```

The flag exits immediately after printing — it does not connect to any cluster
or require any prior setup.

---

## 3. HuggingFace image source

AMK does not pull container images from public registries during install. All
images are bundled into a small set of `*.tar.zst` archives published on the
public HuggingFace dataset:

- **Repository:** [`infineon/amk`](https://huggingface.co/datasets/infineon/amk) (public)
- **Archives downloaded:**
  - `amd64-infra-images.tar.zst` — Kubeflow core, MinIO, MySQL (incl. pinned
    `katib-mysql:8.0.36`), metrics-server, …
  - `amd64-kfp-images.tar.zst` — Kubeflow Pipelines runtime images
  - `amd64-argo-argocd-images.tar.zst` — Argo Workflows + Argo CD
  - `amd64-elyra-images.tar.zst` — Elyra notebook images (with
    `kfp-kubernetes==1.1.0` baked in for KFP 2.4 driver compatibility)
  - `amd64-gpu-images.tar.zst` — NVIDIA device plugin (only when `--gpu`)

| Concern             | Behavior                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Token required?** | **No.** The dataset is public; downloads work anonymously.                                                               |
| **Where cached?**   | `~/.amk/images/` (Linux/macOS) or `%USERPROFILE%\.amk\images\` (Windows). Re-runs reuse the cache.                       |
| **Network needed?** | First install only. Subsequent `--command create` runs reuse the cache and need no internet.                             |
| **Air-gapped use?** | Pre-stage the archives manually into the cache directory above; no other registry access is required.                    |
| **Rate limits?**    | HuggingFace has generous public limits. If you hit them, set `HF_TOKEN` (or `HUGGINGFACE_HUB_TOKEN`) to your free token. |
| **Docker Hub?**     | Not used. No `docker login` required. Missing images fail fast with the HF URL that could not be resolved.               |

### Optional: provide an HF token

If you want higher download throughput or use a private mirror, set the
environment variable **before** running `amk`:

```bash
# Linux
export HF_TOKEN="hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
amk --command create
```

```powershell
# Windows
$env:HF_TOKEN = "hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
amk.exe --command create
```

To point at a private mirror of the dataset, override the repo ID:

```bash
export AMK_HF_REPO_ID="my-org/amk-mirror"
```

> The installer treats HuggingFace as the **single source of truth** for
> container images. No fallback to public registries is performed during
> deploy. If a HuggingFace download fails, the install fails fast with the URL
> that could not be retrieved — fix connectivity and re-run.

---

## 4. Quick start

```bash
# Linux: minimal install, local-disk storage
amk --command create --cluster-name kubeflow --storage-mode local

# Linux: with GPU (requires NVIDIA driver + Container Toolkit on host)
amk --command create --cluster-name kubeflow --storage-mode local --gpu
```

```powershell
# Windows: minimal install
amk.exe --command create --cluster-name kubeflow --storage-mode local

# Windows: with GPU (requires NVIDIA Windows driver + WSL2 GPU + toolkit in Podman VM)
amk.exe --command create --cluster-name kubeflow --storage-mode local --gpu
```

When `create` finishes, the platform is exposed on `http://localhost:9002` via
a single `kubectl port-forward`. Open the URL in a browser. On Windows, when
multiple installations coexist, the second cluster is exposed on `9003`, the
third on `9004`, and so on (override anytime with `--port-forward-port`).

---

## 5. Lifecycle commands

| `--command` | What it does                                                                            |
| ----------- | --------------------------------------------------------------------------------------- |
| `create`    | One-shot install: KIND cluster + Kubeflow + (optionally) GPU. Now waits for the Notebooks subsystem to be ready before returning. |
| `status`    | Print cluster health, node state, and active port-forward(s). Add `--panels` for the full dashboard-routes validation suite. |
| `set`       | Resume after host reboot — restarts containers and re-establishes the port-forward.     |
| `stop`      | Kill the port-forward only. The cluster keeps running.                                  |
| `recover`   | Restart core Kubeflow Deployments when the API is reachable but workloads are degraded. |
| `delete`    | Kill the port-forward and delete the entire KIND cluster.                               |
| `test`      | End-to-end smoke test of every dashboard route (use with `--test-target web-gui`).      |

After a host reboot, simply run:

```bash
amk --command set
```

It will start any stopped KIND containers, kill stale port-forwards, and bring
`http://localhost:9002` back up.

### Validate the dashboard panels (new in 0.1.1)

```bash
amk --command status --panels
```

Walks every left-panel route (Notebooks, TensorBoards, Volumes, Katib,
Pipelines, MLflow, Argo CD, Argo Workflows, Kubernetes Dashboard) plus the
backing Kubernetes services. Exits non-zero on the first failed route, so it
fits cleanly into CI / cron health checks.

---

## 6. Accessing the platform

A single port-forward on `localhost:9002` (default — change with
`--port-forward-port`) exposes everything via a built-in `auth-proxy`:

| URL                                     | What it serves                                                               |
| --------------------------------------- | ---------------------------------------------------------------------------- |
| `http://localhost:9002/`                | Kubeflow Central Dashboard (Notebooks, Pipelines, Katib, Volumes, MLflow, …) |
| `http://localhost:9002/argocd/`         | Argo CD (GitOps)                                                             |
| `http://localhost:9002/argo-workflows/` | Argo Workflows UI                                                            |
| `http://localhost:9002/k8s/`            | Official Kubernetes Dashboard                                                |

Default credentials for the Kubeflow login page:

| Field    | Value              |
| -------- | ------------------ |
| Email    | `user@example.com` |
| Password | `12341234`         |

Single sign-on is wired up so logging in once gives access to all four UIs.

> **MinIO Console is not exposed** (its AGPLv3 license is incompatible with
> commercial use). The MinIO S3 API still runs internally and is used by
> Pipelines and MLflow for artifact storage. To inspect buckets use the `mc`
> CLI or any S3 client against
> `kubectl -n kubeflow port-forward svc/minio-service 9000:9000`.

---

## 7. GPU support (optional)

Pass `--gpu` only when **all** of the following are true:

- You have an NVIDIA GPU physically present (or passed through to WSL2).
- You intend to run workloads that request `nvidia.com/gpu`.
- The host-level GPU stack is already installed and verified
  (driver + NVIDIA Container Toolkit; on Windows: Podman VM has the toolkit
  installed and CDI spec generated).

To pre-validate, on Linux:

```bash
nvidia-smi
nvidia-ctk --version
docker run --rm --gpus all nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi
```

On Windows:

```powershell
wsl -e nvidia-smi
podman machine ssh "nvidia-smi"
podman machine ssh "ls -l /etc/cdi/nvidia.yaml"
```

After installing with `--gpu`, verify GPU is schedulable:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{": gpu="}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
```

You can also enable, disable, or query GPU support on a running cluster
without recreating it:

```bash
amk --command set --gpu-action status
amk --command set --gpu-action enable
amk --command set --gpu-action disable
```

---

## 8. Troubleshooting

| Symptom                                        | Try                                                                                                      |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `command -v podman.exe` missing                | Install Podman Desktop, then run `podman machine init && podman machine start`                           |
| `bash.exe` not found (Windows)                 | Install Git for Windows (provides `C:\Program Files\Git\bin\bash.exe`); make sure it is on `PATH`        |
| HuggingFace download hangs/fails               | Check internet to `huggingface.co`; if behind a proxy, set `HTTPS_PROXY`; optional: set `HF_TOKEN`       |
| `wsl.exe ... exit status 0xffffffff` (Windows) | Transient WSL2 race — the installer auto-retries up to 3×. If it persists: `wsl --shutdown`, then re-run |
| `nvidia.com/gpu` shows `0` after `--gpu`       | Re-check host prerequisites in §7 above; `wsl -e nvidia-smi` must succeed on Windows                     |
| Port `9002` already in use                     | Re-run with `--port-forward-port 9003` (or any free port). On Windows, additional installs already auto-pick the next free port. |
| `status` reports port-forward missing right after `create` | Fixed in 0.1.1 — upgrade if you still see this on the very first `status` after install.       |
| Notebooks tab empty / "image pulling" forever  | 0.1.1 waits for Notebook readiness during `create`; if upgrading from 0.1.0, re-run `amk --command set`. |
| Want a full dashboard sanity check             | `amk --command status --panels`                                                                          |
| Want to reset everything                       | `amk --command delete` removes the KIND cluster; cached images at `~/.amk/images/` are kept for re-use   |

For deeper diagnostics:

```bash
amk --command status                     # cluster + port-forward state
amk --command status --panels            # full dashboard-routes validation
kubectl get pods -A                      # pod-level view
kubectl logs -n kubeflow <pod>           # individual pod logs
```

---

## 9. Uninstall

```bash
amk --command delete
```

This stops the port-forward and removes the KIND cluster (containers, network,
volumes). The HuggingFace image cache at `~/.amk/images/` (or
`%USERPROFILE%\.amk\images\`) is **kept** so the next `create` is fast. To
fully purge, delete that directory manually.

---

## 10. Upgrading from 0.1.0

1. `amk --command delete` on the old install (optional — the 0.1.1 binary is
   fully compatible with a 0.1.0-created cluster, but a clean re-create picks
   up the new pinned `katib-mysql` and Elyra images).
2. Replace the on-disk `amk` binary with the 0.1.1 build of the same OS/CPU.
3. `amk --version` should now report `0.1.1`.
4. `amk --command create ...` (or `amk --command set` to resume an existing
   cluster).
5. Verify with `amk --command status --panels`.

The cached HuggingFace archives in `~/.amk/images/` do **not** need to be
deleted; new images introduced in 0.1.1 are downloaded incrementally on demand.

---

## 11. Version & support

- **Release:** AMK 0.1.1
- **Image source:** [`infineon/amk` on HuggingFace](https://huggingface.co/datasets/infineon/amk)
  — public, no token required.

To print the embedded version at any time:

```bash
amk --version       # 0.1.1
```
