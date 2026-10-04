# Blue-Green Demo — Kargo + Argo CD on Akuity Platform

A minimal, Kubernetes-native blue-green deployment driven by Kargo. No Argo Rollouts: just two Deployments and Service selectors.

## How it works

Two copies of NGINX run side by side: `blue` and `green`. Two Services choose between them by their `color` selector:

- `nginx` (**live**) is what users hit.
- `nginx-preview` (**preview**) points at the color being tested.

Each color also has its own Service (`nginx-blue`, `nginx-green`) for reaching it directly.

| Stage | What it does |
|---|---|
| `preview` | Reads which color is live, deploys the new image to the **other** color, and points `nginx-preview` at it. Users aren't affected. |
| `live` | Points `nginx` at the color running this Freight's tag |

Releases alternate between colors: blue → green → blue. The previous version keeps running on the idle color, so rolling back is just another switch.

```text
Warehouse ─► Freight ─► preview Stage: new tag on idle color, nginx-preview → it ─► live Stage: nginx → that color
```

## Prerequisites

- Akuity account with an Argo CD instance and a Kargo instance
- A Kubernetes cluster (for example [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)) connected with an [Argo CD agent](https://docs.akuity.io/argocd/getting-started/connect-kubernetes-cluster) and a [Kargo agent](https://docs.akuity.io/kargo/getting-started/connect-kargo-agent)
- A copy of this repo in your own GitHub account, and a GitHub PAT with read and write access to it
- CLIs: `akuity`, `kargo`, `kubectl`, [`task`](https://taskfile.dev/docs/installation), `envsubst` (`brew install akuity kargo go-task gettext`)

## Setup

1. Set up `.env`:

   ```bash
   cp .env.example .env
   ```

   Fill in your instance names, repo URL, GitHub credentials and the name of the cluster registered to Argo CD.

   > Never commit `.env`. It contains your PAT.

2. Log in and apply everything:

   ```bash
   akuity login
   task setup
   ```

   Kargo never pushes to `main`. On the first `preview` promotion it creates two branches with the rendered manifests, `stage/preview` and `stage/live`, and the Argo CD apps track those. Until then the apps show "unable to resolve stage/…", which is expected.

3. Promote the **oldest** Freight to `preview` once, to save the newer ones for the demo. This creates the `stage/` branches and deploys blue, green and `nginx-preview`.

4. In the Argo CD UI, sync `bluegreen-live` once. The `preview` Stage isn't allowed to sync it, so the `nginx` Service doesn't exist until you do.

   You now start with **blue** live on `1.27.0`, and green on the Freight from step 3.

## Demo

Open three terminals:

```bash
task port-forward-live      # http://localhost:8080  (what users see)
task port-forward-preview   # http://localhost:8081  (what you're testing)
task watch-status           # which color is live / in preview, and their versions
```

`task watch-status` shows, refreshed every 2 seconds:

```text
LIVE     nginx          → blue   1.27.0
PREVIEW  nginx-preview  → green  1.31.5
```

Each page also shows its color, the NGINX version it runs and the pod name.

To reach a color directly, use `task port-forward-blue` (8082) or `task port-forward-green` (8083).

The steps below start from the state after setup: **blue** is live.

1. **Release to preview.** Promote a newer Freight to `preview`. It deploys to **green** (because blue is live), and `nginx-preview` now points at green:
   - http://localhost:8081 shows **GREEN** with the new NGINX version
   - http://localhost:8080 still shows **BLUE**

2. **Switch live.** Promote the same Freight to `live`. `nginx` now selects green, so http://localhost:8080 shows **GREEN**. Blue keeps running the old version.

3. **Next release.** Promote the next Freight to `preview`. This time it goes to **blue**, because green is live. Promote it to `live` when it looks good.

4. **Roll back.** Promote the previous Freight straight to `live`. It's still running on the other color, so traffic switches back instantly.

Check the commits Kargo pushed. It never touches `main`; each Stage renders plain YAML into its own branch:
- `stage/preview`: each release changes the image in one color's Deployment (`blue/` or `green/`) and the selector in `preview/service-nginx-preview.yaml`
- `stage/live`: each switch is a one-line change to `spec.selector.color` in `live/service-nginx.yaml`

> `kubectl port-forward svc/...` connects to a single pod when it starts, so it doesn't follow a selector change. Restart the port-forward after each promotion. The pages are served with `Cache-Control: no-store`, so a normal refresh is enough.

> To change the app (for example `env/base/`), commit it to `main`. The next `preview` promotion renders it onto the idle color; the color serving users isn't touched until you switch to it.

## Repository Structure

```text
.
├── env/                # Kustomize sources (main); Kargo renders them into stage/preview and stage/live
│   ├── base/           # NGINX Deployment + Service + nginx config, shared by both colors
│   └── overlays/       # one Argo CD app each: bluegreen-<overlay>
│       ├── blue/       # -blue suffix, color label, page, start tag   (rendered by Stage preview)
│       ├── green/      # same for green                               (rendered by Stage preview)
│       ├── preview/    # nginx-preview Service, the color being tested (rendered by Stage preview)
│       └── live/       # nginx Service users hit                      (rendered by Stage live)
├── argocd/             # ApplicationSet generating the four Argo CD apps
├── kargo/              # Project, Warehouse, Stages
├── secret.yaml         # Kargo Git credentials (filled from .env)
├── Taskfile.yaml
└── .env.example
```

## Cleanup

```bash
kargo login https://<your-kargo-instance-url> --sso
argocd login <your-argocd-instance-host> --grpc-web
task cleanup
```

`task cleanup` also deletes the `stage/preview` and `stage/live` branches, so the next setup starts again from `main` (blue live, both colors on `1.27.0`).
