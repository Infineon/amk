# AMK 0.1.2 — Client Installation Guide

AMK ("AI/ML Kubeflow") is a self-contained, single-binary installer that deploys a
local Kubeflow platform on top of [KIND](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker
or Kubernetes-in-Podman). All container images are sourced from the public
HuggingFace dataset [`infineon/amk`](https://huggingface.co/datasets/infineon/amk),
so installation works on machines with restricted access to public registries
(Docker Hub, gcr.io, quay.io, registry.k8s.io, nvcr.io …).

This document is the entry point for **end users** running the binaries shipped
in this release. For developer/build documentation see the source repository.

---

## What's new in 0.1.2

Highlights since 0.1.1:

- **Notebook kernels stay alive ("Kernel died unexpectedly" fixed).** A stack of
  `auth-proxy` and Elyra image fixes makes Jupyter / Elyra notebook kernels
  reliable behind the single `localhost:9002` port-forward:
  - `auth-proxy` now sets `proxy_read_timeout` / `proxy_send_timeout` to
    24 hours for notebook WebSocket routes (was 60 s, which silently killed
    long-lived kernel sockets).
  - `auth-proxy` strips the `Sec-WebSocket-Protocol: v1.kernel.websocket.jupyter.org`
    sub-protocol that newer Jupyter clients negotiate but the kernel gateway
    cannot honor over the proxied path.
  - The Elyra notebook image disables the
    `v1.kernel.websocket.jupyter.org` wire protocol at the server level, so
    even direct in-cluster clients fall back to the supported v0 protocol.
  - `auth-proxy` injects `X-XSRFToken` from the `_xsrf` cookie, fixing CSRF
    rejections on notebook `POST /api/...` calls.
  - `auth-proxy` returns a JSON `401` (not an HTML `302`) for unauthenticated
    `/api/` requests, so JupyterLab surfaces a clear error instead of trying to
    parse the login page as JSON.
- **Notebook readiness webhook restored.** The mutating webhook that gates
  Notebook pods on backing-image readiness (originally introduced in 0.1.1
  daccae6) was reinstated together with an `auth-proxy` 502 guard so the
  dashboard never serves a half-ready Notebooks tab.
- **Native ARM64 images via HuggingFace — no more QEMU.** ARM64 container
  images (Elyra notebook, Kubeflow infra) are now published to the
  `infineon/amk` HuggingFace dataset as native arm64 archives. The installer
  no longer needs `qemu-user-static` / `binfmt_misc` to run on Apple Silicon
  or ARM Linux hosts; ARM64 installs pull native images and skip emulation.
  - New publish helper: `elyra arm64 publish script + sync local-image
    fallback`.
  - The new `arm-elyra` image variant is the default Elyra notebook image on
    `linux/arm64` and `windows/arm64` (Podman) hosts.
- **Elyra image: timing & GPU fixes.**
  - Resolved a startup race where the Elyra extension loaded before the
    JupyterServer was fully up, occasionally leaving the launcher empty.
  - New `elyra-gpu` image variant + `test-elyra-gpu.sh` smoke test exercises
    `nvidia.com/gpu` from inside an Elyra notebook end-to-end.
- **System-wide installation mode (multi-user hosts).** New flags make AMK
  installable as a shared service that all users on a host can use:
  - `--system-wide` — installs into a shared location instead of the
    invoking user's home.
  - `--shared-root <path>` — overrides the shared root directory; defaults to
    `/srv/amk`. Container images, KIND state, and helper scripts live here so
    each user does not re-download the same archives.
  - Per-user runtime files (kubeconfig context, port-forward PID) remain under
    `~/.amk/` so multiple users can `amk --command set` independently against
    the same shared cluster.
- **Side-by-side install documentation.** New docs cover running multiple AMK
  versions on the same host (binary placement, port allocation, image-cache
  isolation) and re-building from source for contributors.
- **Multi-user / multi-developer workflow docs.** Step-by-step guidance for
  teams sharing a single `--system-wide` cluster:
  - Each developer keeps their own `LPWWD_REPO_ROOT` checkout (no shared
    working tree).
  - Shared read-only data is exposed via a relative symlink at
    `shared/data` instead of bind-mounting absolute paths, so the same
    notebook works for every user.
  - The application repo is cloned at the project root, **not** inside
    `shared/core/`, to keep the shared tree clean.
  - Includes a copy-pasteable validation snippet to verify the layout before
    launching pipelines.
- **`--command status --panels` restored (Windows regression fix).** The
  panels-validation suite shipped in 0.1.1 had regressed on Windows in early
  0.1.2 builds. It now works again on all four supported binaries, with two
  additional improvements:
  - Auto-detects the `auth-proxy` `BASE_URL` (no need to pass
    `--port-forward-port` when running against a non-default port).
  - Defaults `TEST_NAMESPACE` to the invoking user's Kubeflow Profile
    (`user@example.com` → `kubeflow-user-example-com`), matching the default
    Profile created by `create`.
  - Adds a notepad-kernel ("nootepad test") panel that opens a notebook and
    executes a kernel cell, end-to-end-validating the WebSocket fixes above.
- **Windows: notebook-readiness webhook fix.** A Windows-specific bug where
  the notebook-readiness webhook URL was rewritten incorrectly through the
  WSL2 / Podman networking stack has been corrected, so `create` on Windows
  now waits on the same readiness signal as Linux.

The command-line surface remains backwards compatible with 0.1.1; all new
behavior is opt-in via new flags (`--system-wide`, `--shared-root`).

---

## 1. Pick the right binary

The release ships four cross-compiled binaries. Pick the one matching your
machine and ignore the others.

| File                    | OS      | CPU    | Container runtime | Size    |
| ----------------------- | ------- | ------ | ----------------- | ------- |
| `amk-linux-amd64`       | Linux   | x86_64 | Docker            | ~9.3 MB |
| `amk-linux-arm64`       | Linux   | ARM64  | Docker            | ~8.7 MB |
| `amk-windows-amd64.exe` | Windows | x86_64 | Podman            | ~9.4 MB |
| `amk-windows-arm64.exe` | Windows | ARM64  | Podman            | ~8.7 MB |

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
amk --version       # prints: 0.1.2
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

> **ARM64 Linux:** `qemu-user-static` / `binfmt_misc` are no longer required —
> 0.1.2 ships native arm64 images from HuggingFace.

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
- **Archives downloaded (per architecture):**
  - `<arch>-infra-images.tar.zst` — Kubeflow core, MinIO, MySQL (incl. pinned
    `katib-mysql:8.0.36`), metrics-server, …
  - `<arch>-kfp-images.tar.zst` — Kubeflow Pipelines runtime images
  - `<arch>-argo-argocd-images.tar.zst` — Argo Workflows + Argo CD
  - `<arch>-elyra-images.tar.zst` — Elyra notebook images, including the new
    `elyra-gpu` variant; arm64 builds ship the `arm-elyra` image (no QEMU).
  - `<arch>-gpu-images.tar.zst` — NVIDIA device plugin (only when `--gpu`)

  `<arch>` is `amd64` or `arm64`; the installer picks the right set
  automatically.

| Concern             | Behavior                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Token required?** | **No.** The dataset is public; downloads work anonymously.                                                               |
| **Where cached?**   | `~/.amk/images/` (Linux/macOS) or `%USERPROFILE%\.amk\images\` (Windows). With `--system-wide`, cache lives under `--shared-root` (default `/srv/amk/images/`) and is reused by every user on the host. |
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

# Linux: shared system-wide install (root once; users 'set' afterwards)
sudo amk --command create --cluster-name kubeflow --storage-mode local \
         --system-wide --shared-root /srv/amk
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
| `create`    | One-shot install: KIND cluster + Kubeflow + (optionally) GPU. Now also accepts `--system-wide` / `--shared-root` for multi-user hosts. |
| `status`    | Print cluster health, node state, and active port-forward(s). Add `--panels` for the full dashboard-routes validation suite (now includes a notebook-kernel test). |
| `set`       | Resume after host reboot — restarts containers and re-establishes the port-forward. Per-user when run against a `--system-wide` cluster. |
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

### Validate the dashboard panels

```bash
amk --command status --panels
```

Walks every left-panel route (Notebooks, TensorBoards, Volumes, Katib,
Pipelines, MLflow, Argo CD, Argo Workflows, Kubernetes Dashboard) plus the
backing Kubernetes services, **and** opens a notebook to execute a kernel cell
end-to-end (verifies the 0.1.2 WebSocket / XSRF / timeout fixes). Exits
non-zero on the first failed route, so it fits cleanly into CI / cron health
checks. The suite now auto-detects the `auth-proxy` base URL and the user's
Kubeflow Profile namespace.

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

The new `elyra-gpu` notebook image (auto-selected when `--gpu` is on) gives
notebook authors `torch` / `tensorflow` GPU access out of the box; verify with
`scripts/test-elyra-gpu.sh`.

---

## 8. System-wide / multi-user installs (new in 0.1.2)

For shared workstations where multiple developers want to share a single
KIND cluster and a single image cache:

```bash
sudo amk --command create \
         --cluster-name kubeflow \
         --storage-mode local \
         --system-wide \
         --shared-root /srv/amk
```

What changes vs. a per-user install:

- Container images, KIND state, and helper scripts live under
  `--shared-root` (default `/srv/amk`) instead of `~/.amk/`.
- Each user runs `amk --command set` from their own account to create a
  per-user kubeconfig context and start their own `kubectl port-forward`
  (default port still `9002`; collisions auto-bump to `9003`, `9004`, …).
- Per-developer LPWWD-style workflows: each user keeps their own
  `LPWWD_REPO_ROOT` checkout (clone the application repo at the project root,
  **not** inside `shared/core/`). Shared read-only datasets are exposed at
  `shared/data` via a relative symlink so the same notebook works in every
  user's checkout.

See the side-by-side and multi-user docs in the source repository for
copy-pasteable layouts and a validation snippet.

---

## 9. Troubleshooting

| Symptom                                                    | Try                                                                                                      |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `command -v podman.exe` missing                            | Install Podman Desktop, then run `podman machine init && podman machine start`                           |
| `bash.exe` not found (Windows)                             | Install Git for Windows (provides `C:\Program Files\Git\bin\bash.exe`); make sure it is on `PATH`        |
| HuggingFace download hangs/fails                           | Check internet to `huggingface.co`; if behind a proxy, set `HTTPS_PROXY`; optional: set `HF_TOKEN`       |
| `wsl.exe ... exit status 0xffffffff` (Windows)             | Transient WSL2 race — the installer auto-retries up to 3×. If it persists: `wsl --shutdown`, then re-run |
| `nvidia.com/gpu` shows `0` after `--gpu`                   | Re-check host prerequisites in §7 above; `wsl -e nvidia-smi` must succeed on Windows                     |
| Port `9002` already in use                                 | Re-run with `--port-forward-port 9003` (or any free port). On Windows, additional installs already auto-pick the next free port. |
| **"Kernel died unexpectedly" in Jupyter / Elyra**          | Fixed in 0.1.2 (`auth-proxy` 24h WS timeouts + `Sec-WebSocket-Protocol` strip + image-side WS-protocol disable). Upgrade if you still see this. |
| Notebook `POST /api/...` returning HTML / 302              | Fixed in 0.1.2: `auth-proxy` returns JSON 401 and injects `X-XSRFToken`. Upgrade and hard-refresh.       |
| ARM64 install: `exec format error` / very slow image loads | Fixed in 0.1.2: native arm64 images shipped via HF, no QEMU needed. Re-run `create` after upgrading.     |
| `--command status --panels` errors on Windows              | Fixed in 0.1.2 (Windows regression vs. 0.1.1).                                                           |
| Notebooks tab empty / "image pulling" forever              | 0.1.2 restores the readiness webhook from 0.1.1 daccae6 + 502 guard. Re-run `amk --command set`.         |
| Want a full dashboard sanity check                         | `amk --command status --panels`                                                                          |
| Want to reset everything                                   | `amk --command delete` removes the KIND cluster; cached images are kept for re-use                       |

For deeper diagnostics:

```bash
amk --command status                     # cluster + port-forward state
amk --command status --panels            # full dashboard-routes + kernel validation
kubectl get pods -A                      # pod-level view
kubectl logs -n kubeflow <pod>           # individual pod logs
```

---

## 10. Uninstall

```bash
amk --command delete
```

This stops the port-forward and removes the KIND cluster (containers, network,
volumes). The HuggingFace image cache at `~/.amk/images/` (or
`%USERPROFILE%\.amk\images\`, or `<shared-root>/images/` for `--system-wide`
installs) is **kept** so the next `create` is fast. To fully purge, delete
that directory manually.

---

## 11. Upgrading from 0.1.1

1. `amk --command delete` on the old install (optional — the 0.1.2 binary is
   fully compatible with a 0.1.1-created cluster, but a clean re-create picks
   up the new Elyra / `auth-proxy` images and the native arm64 images).
2. Replace the on-disk `amk` binary with the 0.1.2 build of the same OS/CPU.
3. `amk --version` should now report `0.1.2`.
4. `amk --command create ...` (or `amk --command set` to resume an existing
   cluster).
5. Verify with `amk --command status --panels` — the suite now also opens a
   notebook and executes a kernel cell to confirm the WebSocket / XSRF fixes.

The cached HuggingFace archives in `~/.amk/images/` do **not** need to be
deleted; new images introduced in 0.1.2 (`arm-elyra`, `elyra-gpu`, refreshed
`auth-proxy`) are downloaded incrementally on demand. ARM64 hosts will fetch
the new native arm64 archive set on first `create` after upgrading.

If you previously relied on `qemu-user-static` / `binfmt_misc` to run AMK on
an ARM host, you can remove them — 0.1.2 no longer needs emulation.

---

## 12. Version & support

- **Release:** AMK 0.1.2
- **Image source:** [`infineon/amk` on HuggingFace](https://huggingface.co/datasets/infineon/amk)
  — public, no token required.

To print the embedded version at any time:

```bash
amk --version       # 0.1.2
```
