# LPWWD multi-user setup with a shared `data/` tree

How to share the heavy `shared/data/` (negative_data ~31 GB, noise_rir
~8.7 GB) across users of a system-wide AMK cluster while keeping each user's
project tree (code edits, keyword configs, training output) private.

> **Status:** the upstream code already supports this; no source changes
> needed. Only filesystem layout, group ownership, and a one-time cluster
> reinstall on a shared `HOST_AMK_ROOT`.

---

## What we already have in 0.1.1

Layout already separates shared from per-user data cleanly:

```
/home/alex/amk/projects/lpwwd/
├── shared/
│   ├── core/     # code (per user; only the heavy data is truly shared)
│   └── data/     # 39 GB, cross-user (negative_data 31G, noise_rir 8.7G)
└── keywords/
    └── hey_deepcraft/
        ├── data/   # per-keyword configs + positive_data (small, per-user)
        └── output/ # per-user training output (large, per-user)
```

…and four env vars already route the pipeline at the right roots
(commit *"paths: wire env-var defaults + LPWWD_SHARED_DATA_ROOT for
Option-D1 layout"*):

| Var                      | Purpose                              | Per-user or shared?                           |
| ------------------------ | ------------------------------------ | --------------------------------------------- |
| `LPWWD_REPO_ROOT`        | code (`shared/core/`)                | **per-user** (each dev has their own checkout) |
| `LPWWD_SHARED_DATA_ROOT` | `negative_data` + `noise_rir`        | **shared (RW for `amk-users` group)**         |
| `LPWWD_DATA_ROOT`        | per-keyword config + `positive_data` | per-user                                      |
| `LPWWD_OUTPUT_ROOT`      | training output                      | per-user                                      |

> The directory is named `shared/core/` for historical reasons, but only
> `shared/data/` is actually shared across users. Each developer keeps
> their own clone of `shared/core/` so they can edit/branch independently;
> trying to share a single working copy would force everyone onto the
> same git branch and break per-user pipelines.

So the design problem is solved. Only the *filesystem layout* needs to
reflect it.

---

## env vars, no symlinks

Cleanest because:

- Symlinks crossing into another user's `$HOME` still need traversal
  perms on `/home/alex` (Ubuntu default `0750` blocks it).
- Symlinks inside the kind mount work, but obscure where data really lives.
- You can swap the shared root without moving files.

### 1. One-time host setup (as root)

Only `shared/data/` (the 39 GB of negative+noise audio) is moved to the
shared root. Code (`shared/core/`) stays per-user, cloned by each developer
into their own subtree.

```bash
sudo groupadd -f amk-users
sudo usermod -aG amk-users alex
sudo usermod -aG amk-users bob

# Shared root, group-writable, sgid so new files inherit the group
sudo install -d -m 2775 -g amk-users /srv/amk
sudo install -d -m 2775 -g amk-users /srv/amk/projects
sudo install -d -m 2775 -g amk-users /srv/amk/shared
sudo install -d -m 2775 -g amk-users /srv/amk/shared/data

# Move the heavy bits there once (data only — NOT shared/core/)
sudo rsync -aHAX --info=progress2 \
    /home/alex/amk/projects/lpwwd/shared/data/ \
    /srv/amk/shared/data/

sudo chgrp -R amk-users /srv/amk/shared/data
sudo chmod -R g+rwX,o-rwx /srv/amk/shared/data
sudo find /srv/amk/shared/data -type d -exec chmod g+s {} +
```

> `/srv/amk/shared/data/` is the **only** truly shared directory. Each
> user's project tree lives at `/srv/amk/projects/<user>-lpwwd/` (see
> step 3) and contains their own clone of `shared/core/`.

### 2. Reinstall the cluster on the shared root

The kind cluster bind-mounts a single host directory at `/mnt/amk` inside
every pod. That mount is baked in at `kind create cluster` time, so
changing it requires a delete + create.

**With AMK installer ≥ 0.1.2 (`--shared-root` flag, `/srv/amk` default):**

```bash
cd /home/alex/amk-deploy/go/amk-universal-installer/dist
sudo -E ./amk --command delete --system-wide
sudo -E ./amk --command create --system-wide          # picks up /srv/amk by default
# or be explicit:
sudo -E ./amk --command create --system-wide --shared-root /srv/amk
```

**With older installers (env-var fallback, still supported):**

```bash
sudo -E HOST_AMK_ROOT=/srv/amk ./amk --command create --system-wide
```

Now `/mnt/amk` inside every pod = `/srv/amk` on the host.

> Anything written *inside* the cluster (PVCs, MLflow runs, KFP DB,
> notebook PVCs) is gone after the reinstall. Stuff under the new
> `HOST_AMK_ROOT` survives because it lives on the host.

### 3. Per-user project setup

Each user gets their **own clone of the code** under their personal
project dir on the shared mount, plus their own per-keyword data/output
subtrees. Only `LPWWD_SHARED_DATA_ROOT` points into the cross-user pool.

```bash
# bob's setup (run as bob; sudo only for the install -d)
sudo install -d -m 2750 -g bob /srv/amk/projects/bob-lpwwd

# IMPORTANT: clone INTO the project dir, not into shared/core/.
# The repo's top level already contains shared/, keywords/, .gitignore,
# .lpwwd-keyword, etc. — the project tree IS the repo root. Cloning
# into shared/core/ would give you a nested shared/core/shared/core/.
sudo -u bob git clone <lpwwd-repo-url> /srv/amk/projects/bob-lpwwd

# Hook the cross-user audio pool into bob's tree.
# IMPORTANT: use a *relative* target. An absolute /srv/amk/... target
# would be dangling inside kind pods, where the host's /srv/amk is
# bind-mounted as /mnt/amk and /srv/amk does not exist.
rm -rf /srv/amk/projects/bob-lpwwd/shared/data   # remove the placeholder dir from the clone
ln -sfn ../../../shared/data /srv/amk/projects/bob-lpwwd/shared/data

# (Optional) per-keyword output dir is gitignored — the clone only
# ships its README.md placeholder; create the dir on demand.
mkdir -p /srv/amk/projects/bob-lpwwd/keywords/hey_deepcraft/output

# Convenience symlink so familiar host paths still work
ln -s /srv/amk/projects/bob-lpwwd ~/amk/projects/bob-lpwwd

cat >> ~/.bashrc <<'EOF'
# AMK / LPWWD
export HOST_AMK_ROOT=/srv/amk                                      # for kind helper scripts
export LPWWD_PROJECT_NAME=$USER-lpwwd
# In-pod paths (everything is under /mnt/amk = /srv/amk on the host).
export LPWWD_REPO_ROOT=/mnt/amk/projects/$USER-lpwwd/shared/core         # subdir of the clone, holds python source
export LPWWD_SHARED_DATA_ROOT=/mnt/amk/shared/data                       # cross-user, RW group
export LPWWD_DATA_ROOT=/mnt/amk/projects/$USER-lpwwd/keywords/hey_deepcraft/data
export LPWWD_OUTPUT_ROOT=/mnt/amk/projects/$USER-lpwwd/keywords/hey_deepcraft/output
EOF
```

Note: `configs/` and `positive_data/` come down with the clone already —
they're tracked under `keywords/<kw>/data/` at the repo top level, which
is exactly where `LPWWD_DATA_ROOT` points. No copy or symlink needed for
the per-user case.

Validation (host + in-pod) one-liner:

```bash
bash -lc '
  for p in $LPWWD_REPO_ROOT $LPWWD_SHARED_DATA_ROOT $LPWWD_DATA_ROOT $LPWWD_OUTPUT_ROOT; do
    docker exec $(docker ps --format "{{.Names}}" | grep control-plane | head -1) \
      sh -c "test -e $p && echo \"OK   $p\" || echo \"MISS $p\""
  done'
```

Resulting layout on the host:

```
/srv/amk/                                  # = /mnt/amk inside pods
├── shared/
│   └── data/                          # cross-user (39 GB, group: amk-users, RW)
│       ├── negative_data/
│       └── noise_rir/
└── projects/
    ├── alex-lpwwd/                    # alex's checkout + outputs
    │   ├── shared/core/               # alex's clone (his branch, his edits)
    │   └── keywords/hey_deepcraft/
    │       ├── data/
    │       └── output/
    └── bob-lpwwd/                     # bob's checkout + outputs
        ├── shared/core/               # bob's clone (his own branch, edits)
        └── keywords/hey_deepcraft/
            ├── data/
            └── output/
```

This is the cleanest pattern: **one shared kind mount root, per-user
project subtree (including the code clone) under it, env vars route the
pipeline.** No symlinks crossing user homes, no shared-branch conflicts,
no cluster reinstall when you add a user.

---

## Notes & gotchas

- **UIDs inside pods.** Notebook pods run as a fixed UID (typically `1000`,
  `jovyan`). Files written from inside the cluster are owned by that UID
  on the host regardless of which host user submitted the workflow. The
  shared-group approach (`g+s` on `/srv/amk`, plus a permissive umask in
  pod images) is the usual mitigation.
- **`sudo -E` matters.** The Go installer wrapper passes the environment
  through to the embedded shell script. Without `-E`, `HOST_AMK_ROOT`
  would be stripped and the install would fall back to the built-in
  default (`/srv/amk` for `--system-wide`, `$HOME/amk` for per-user).
  When you use the new `--shared-root` flag the value is injected
  programmatically, so `-E` is no longer required for that one variable
  — but other env vars you may rely on (e.g. `LPWWD_*_ROOT`) still need it.
- **Per-user kubeconfig is unnecessary.** The system-wide install already
  drops `/etc/profile.d/amk.sh` exporting `KUBECONFIG=/etc/amk/kube/config`
  for every login shell, so all users get cluster access automatically.
