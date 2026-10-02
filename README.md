# Blue-Green Demo — Kargo + Argo CD on Akuity Platform

A minimal, Kubernetes-native blue-green deployment driven by Kargo. No Argo Rollouts: just two Deployments and Service selectors.

## How it works

Two copies of NGINX run side by side: `blue` and `green`, each with its own Service (`nginx-blue`, `nginx-green`). A third Service, `nginx`, is what users hit: it points at one color at a time.

| Stage | What it does |
|---|---|
| `preview` | Reads which color is live and deploys the new image to the **other** color. Users aren't affected; test it on that color's own Service. |
| `live` | Points the `nginx` Service at the color running this Freight's tag |

Releases alternate between colors: blue → green → blue. The previous version keeps running on the idle color, so rolling back is just another switch.

```text
Warehouse ─► Freight ─► preview Stage: new tag on idle color ─► live Stage: nginx → that color
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

3. In the Argo CD UI, sync the three apps (`bluegreen-blue`, `bluegreen-green`, `bluegreen-live`) once. At the start, **blue** is live and both colors run `1.27.0`.

## Demo

Open three terminals:

```bash
task port-forward-live    # http://localhost:8080  (what users see)
task port-forward-blue    # http://localhost:8081
task port-forward-green   # http://localhost:8082
```

1. **Release to preview.** Promote a newer Freight to `preview`. It deploys to **green** (because blue is live):
   - http://localhost:8082 shows **GREEN**, and `curl -sI localhost:8082 | grep Server` shows the new NGINX version
   - http://localhost:8080 still shows **BLUE**

2. **Switch live.** Promote the same Freight to `live`. The `nginx` Service now selects green, so http://localhost:8080 shows **GREEN**. Blue keeps running the old version.

3. **Next release.** Promote the next Freight to `preview`. This time it goes to **blue**, because green is live. Promote it to `live` when it looks good.

4. **Roll back.** Promote the previous Freight straight to `live`. It's still running on the other color, so traffic switches back instantly.

Check the commits on `main`: each release changes `newTag` in one color's overlay (`env/overlays/<color>/kustomization.yaml`), and each switch is a one-line change to `spec.selector.color` in `env/overlays/live/service.yaml`.

> `kubectl port-forward svc/...` connects to a single pod when it starts, so it doesn't follow a selector change. Restart `port-forward-live` after each switch.

## Repository Structure

```text
.
├── env/
│   ├── base/           # NGINX Deployment + Service, shared by both colors
│   └── overlays/       # one Argo CD app each: bluegreen-<overlay>
│       ├── blue/       # -blue suffix, color label, page, image tag   (updated by Stage preview)
│       ├── green/      # same for green                               (updated by Stage preview)
│       └── live/       # nginx Service users hit                      (updated by Stage live)
├── argocd/             # the three Argo CD Applications
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
