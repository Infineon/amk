# AMK — Free and Open Source Software (FOSS) Report

**Product:** AMK ("AI/ML Kubeflow") universal installer
**Release:** 0.1.2

This report lists the Free and Open Source Software (FOSS) that AMK
**redistributes or deploys** on the user's machine. It is provided to satisfy
the attribution/notice obligations of the licenses below.

AMK is a self-contained installer binary that provisions a local
[KIND](https://kind.sigs.k8s.io/) cluster and deploys a Kubeflow-based
platform. The bulk of the FOSS listed here is delivered as **container images**
pulled from the public HuggingFace dataset
[`infineon/amk`](https://huggingface.co/datasets/infineon/amk) at install time,
not compiled into the `amk` binary itself.

> **How this list was produced.** Component names and versions were extracted
> directly from the shipped `amk` binary (embedded image references and
> manifest URLs). Licenses reflect the upstream project's license at the pinned
> version. This report is informational and not legal advice; consult each
> upstream project for the authoritative license text.

---

## 1. The `amk` installer binary

| Component | Version | License | Notes |
| --------- | ------- | ------- | ----- |
| `amk` installer (this project) | 0.1.2 | See [`EULA.txt`](EULA.txt) / [`LICENSE.txt`](LICENSE.txt) | Infineon-authored. Docs licensed CC BY-NC 4.0. |
| Go runtime & standard library | go1.26.x | BSD-3-Clause | Statically linked; `CGO_ENABLED=0`. No third-party Go modules are vendored. |

---

## 2. Copyleft components — please read

Some deployed components are under **strong copyleft** licenses. They are used
**unmodified** and run as isolated network services inside the cluster; AMK
does not statically link them or create derivative works.

| Component | Version | License | Handling |
| --------- | ------- | ------- | -------- |
| MinIO (object storage server) | `RELEASE.2023-06-02T23-17-26Z` | GNU AGPL-3.0 | Used internally as the S3 backend for Pipelines/MLflow artifacts. **The MinIO Console UI is intentionally NOT exposed** to avoid AGPL network-use obligations for the UI. |
| MinIO Client (`mc`) | latest (pinned in archive) | GNU AGPL-3.0 | CLI utility image only. |
| MySQL Server | 8.0.29, 8.0.36 | GPL-2.0 (with the Universal FOSS Exception) | Metadata store for Kubeflow / Katib. Used unmodified. |
| BusyBox | 1.35 | GPL-2.0 | Init/utility container. Used unmodified. |
| `python:*-slim` base OS | Debian bookworm/bullseye | GPL-2.0 and other Debian licenses | Debian base layers under various licenses; used unmodified. |

---

## 3. Kubeflow platform (Apache-2.0)

All components below are licensed under the **Apache License 2.0**.

| Component | Image | Version |
| --------- | ----- | ------- |
| Kubeflow Central Dashboard | `kubeflownotebookswg/centraldashboard` | v1.8.0 |
| Notebook Controller | `kubeflownotebookswg/notebook-controller` | v1.9.0 |
| Profile Controller | `kubeflownotebookswg/profile-controller` | v1.8.0, v1.9.2 |
| Access Management (KFAM) | `kubeflownotebookswg/kfam` | v1.8.0, v1.9.2 |
| Jupyter Web App | `kubeflownotebookswg/jupyter-web-app` | v1.9.0 |
| Volumes Web App | `kubeflownotebookswg/volumes-web-app` | v1.9.0 |
| TensorBoards Web App | `kubeflownotebookswg/tensorboards-web-app` | v1.9.0 |
| Metadata Frontend | `kubeflownotebookswg/metadata-frontend` | v1.8.0 |
| Metadata Frontend (legacy) | `kubeflow-images-public/metadata-frontend` | v0.1.8 |
| ML Metadata store server | `tfx-oss-public/ml_metadata_store_server` | 1.14.0 |

### Kubeflow Pipelines (Apache-2.0)

| Component | Image | Version |
| --------- | ----- | ------- |
| API Server | `ml-pipeline/api-server` | 1.8.5, 2.0.0, 2.0.5 |
| Frontend | `ml-pipeline/frontend`, `ghcr.io/kubeflow/kfp-frontend` | 1.8.5 / 2.4.0 |
| Cache Server | `ml-pipeline/cache-server` | 1.8.5 |
| Persistence Agent | `ml-pipeline/persistenceagent` | 2.0.0 |
| Viewer CRD Controller | `ml-pipeline/viewer-crd-controller` | 1.8.5 |
| Visualization Server | `ml-pipeline/visualization-server` | 1.8.5 |
| KFP Driver | `ml-pipeline/kfp-driver` | 2.0.0 |
| KFP Launcher | `ml-pipeline/kfp-launcher` | 2.0.0 |

### Kubeflow Katib (Apache-2.0)

| Component | Image | Version |
| --------- | ----- | ------- |
| Katib UI | `kubeflowkatib/katib-ui` | v0.17.0 |
| Katib Controller | `katib-controller` | v0.17.0 |

### Notebook / Elyra images (Apache-2.0; bundle BSD-3-Clause Jupyter + others)

| Component | Image | Version |
| --------- | ----- | ------- |
| Elyra notebook (AMK build) | `amk/jupyter-elyra` | v1.9.0 (`-amd64` / `-arm64`) |
| Jupyter SciPy | `kubeflownotebookswg/jupyter-scipy` | v1.9.0 |
| Jupyter PyTorch | `kubeflownotebookswg/jupyter-pytorch-full` | v1.9.0 |
| Jupyter PyTorch (CUDA) | `kubeflownotebookswg/jupyter-pytorch-cuda-full` | v1.9.0 |
| Jupyter TensorFlow | `kubeflownotebookswg/jupyter-tensorflow-full` | v1.9.0 |

> These notebook images aggregate many independently licensed Python packages
> (NumPy, pandas, PyTorch, TensorFlow, JupyterLab, Elyra, etc.). Refer to each
> image's own bundled license manifest for the complete per-package list.

---

## 4. Platform infrastructure

| Component | Image | Version | License |
| --------- | ----- | ------- | ------- |
| Argo Workflows | `argoproj/argo-workflows` (manifests) | v3.5.5 | Apache-2.0 |
| Argo CD | `argoproj/argo-cd` (manifests) | v2.9.3 | Apache-2.0 |
| Dex (OIDC provider) | `ghcr.io/dexidp/dex` | v2.37.0 | Apache-2.0 |
| OAuth2 Proxy | `quay.io/oauth2-proxy/oauth2-proxy` | v7.6.0 | MIT |
| Envoy Proxy | `envoyproxy/envoy` | v1.25.0 | Apache-2.0 |
| Kubernetes Dashboard | `kubernetesui/dashboard` | v2.7.0 | Apache-2.0 |
| Dashboard Metrics Scraper | `kubernetesui/metrics-scraper` | v1.0.8 | Apache-2.0 |
| Metrics Server | `registry.k8s.io/metrics-server/metrics-server` | v0.7.2 | Apache-2.0 |
| MLflow | `ghcr.io/mlflow/mlflow` | v2.10.0 | Apache-2.0 |
| KIND (`kindest/node`) | provisioned at runtime | — | Apache-2.0 |
| local-path-provisioner (Rancher) | KIND default storage | — | Apache-2.0 |
| huggingface_hub (image downloader) | host `python3` package | — | Apache-2.0 |
| Zstandard (`.tar.zst` archives) | archive format | — | BSD-3-Clause / GPL-2.0 (dual) |

### Permissively licensed service dependencies

| Component | Image | Version | License |
| --------- | ----- | ------- | ------- |
| Redis | `redis:7-alpine` | 7.x | BSD-3-Clause |
| nginx | `nginx:1.25-alpine` | 1.25 | BSD-2-Clause |

---

## 5. Optional GPU components (`--gpu` only)

Deployed **only** when the installer is run with `--gpu`.

| Component | Image | Version | License |
| --------- | ----- | ------- | ------- |
| NVIDIA Kubernetes Device Plugin | `nvcr.io/nvidia/k8s-device-plugin` | v0.18.2 | Apache-2.0 |
| NVIDIA CUDA base image | `nvidia/cuda:12.3.1-base-ubuntu22.04` | 12.3.1 | **Proprietary — NVIDIA Deep Learning Container License (NOT FOSS)** |

> The NVIDIA CUDA base image is referenced only for host GPU verification /
> GPU-enabled notebook builds. It is governed by NVIDIA's own EULA, not an
> open source license.

---

## 6. Host prerequisites (NOT redistributed)

The following tools must already be present on the host `PATH`. AMK **shells
out** to them but does **not** bundle or redistribute them; they are listed
here only for completeness. Obtain them from their upstream projects under
their respective licenses.

| Tool | Upstream | Typical license |
| ---- | -------- | --------------- |
| Docker / Podman | docker.com / podman.io | Apache-2.0 |
| kind | kind.sigs.k8s.io | Apache-2.0 |
| kubectl | kubernetes.io | Apache-2.0 |
| Python 3 | python.org | PSF License |
| bash / Git for Windows | gnu.org / git-scm.com | GPL-3.0 / GPL-2.0 |

---

## 7. Obtaining source code

Source code for the FOSS components above is available from each project's
upstream repository (GitHub / the registry namespace shown in the tables). For
components AMK is obligated to provide source for, contact Infineon per the
"Free and Open Source Software" clause in [`EULA.txt`](EULA.txt).
