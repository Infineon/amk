# AMK 0.1.0 — Client Installation Guide

AMK ("AI/ML Kubeflow") is a self-contained, single-binary installer that deploys a
local Kubeflow platform on top of [KIND](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker
or Kubernetes-in-Podman). All container images are sourced from the public
HuggingFace dataset [`infineon/amk`](https://huggingface.co/datasets/infineon/amk),
so installation works on machines with restricted access to public registries
(Docker Hub, gcr.io, quay.io, registry.k8s.io, nvcr.io …).

This document is the entry point for **end users** running the binaries shipped
in this release. For developer/build documentation see the source repository.

---

## 1. Pick the right binary

The release ships four cross-compiled binaries. Pick the one matching your
machine and ignore the others.

| File                    | OS      | CPU    | Container runtime | Size    |
| ----------------------- | ------- | ------ | ----------------- | ------- |
| `amk-linux-amd64`       | Linux   | x86_64 | Docker            | ~7.5 MB |
| `amk-linux-arm64`       | Linux   | ARM64  | Docker            | ~7.1 MB |
| `amk-windows-amd64.exe` | Windows | x86_64 | Podman            | ~7.6 MB |
| `amk-windows-arm64.exe` | Windows | ARM64  | Podman            | ~7.1 MB |

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
  - `amd64-infra-images.tar.zst` — Kubeflow core, MinIO, MySQL, metrics-server, …
  - `amd64-kfp-images.tar.zst` — Kubeflow Pipelines runtime images
  - `amd64-argo-argocd-images.tar.zst` — Argo Workflows + Argo CD
  - `amd64-elyra-images.tar.zst` — Elyra notebook images
  - `amd64-gpu-images.tar.zst` — NVIDIA device plugin (only when `--gpu`)

| Concern             | Behavior                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Token required?** | **No.** The dataset is public; downloads work anonymously.                                                               |
| **Where cached?**   | `~/.amk/images/` (Linux/macOS) or `%USERPROFILE%\.amk\images\` (Windows). Re-runs reuse the cache.                       |
| **Network needed?** | First install only. Subsequent `--command create` runs reuse the cache and need no internet.                             |
| **Air-gapped use?** | Pre-stage the archives manually into the cache directory above; no other registry access is required.                    |
| **Rate limits?**    | HuggingFace has generous public limits. If you hit them, set `HF_TOKEN` (or `HUGGINGFACE_HUB_TOKEN`) to your free token. |

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
a single `kubectl port-forward`. Open the URL in a browser.

---

## 5. Lifecycle commands

| `--command` | What it does                                                                            |
| ----------- | --------------------------------------------------------------------------------------- |
| `create`    | One-shot install: KIND cluster + Kubeflow + (optionally) GPU.                           |
| `status`    | Print cluster health, node state, and active port-forward(s).                           |
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
| Port `9002` already in use                     | Re-run with `--port-forward-port 9003` (or any free port)                                                |
| Want to reset everything                       | `amk --command delete` removes the KIND cluster; cached images at `~/.amk/images/` are kept for re-use   |

For deeper diagnostics:

```bash
amk --command status                     # cluster + port-forward state
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

## 10. Version & support

- **Release:** AMK 0.1.0
- **Installer binaries:** built from
  [`amk-deploy`](https://github.com/Infineon/amk-deploy) — see the project
  source for full developer documentation, GPU runbooks, and architecture
  diagrams.
- **Image source:** [`infineon/amk` on HuggingFace](https://huggingface.co/datasets/infineon/amk)
  — public, no token required.

To print the embedded version at any time:

```bash
amk --version
```
