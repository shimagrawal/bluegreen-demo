# CLAUDE.md

Kubernetes-native blue-green demo driven by Kargo + Argo CD on the Akuity Platform. Companion to the `akuity-workshop` repo (`../akuity-workshop`), which uses the same `.env` + `envsubst` + Taskfile conventions.

## Working rules

- Don't commit or push without asking first. The user pushes manually.
- Never commit `.env` (it holds a GitHub PAT).

## Design

- **Rendered branches** (since 2026-10-02). `main` holds Kustomize sources; Kargo never pushes to it. Each Stage renders with `kustomize-build` into its own branch as plain YAML, one file per resource (Kargo naming `kind-name.yaml`, e.g. `deployment-nginx-blue.yaml`):
  - `stage/preview`: `blue/`, `green/`, `preview/`, written only by Stage `preview`.
  - `stage/live`: `live/`, written only by Stage `live` (plus the one-time bootstrap below).
  - Kargo creates both branches (`git-clone` `create: true`); no setup step. The ApplicationSet uses `targetRevision: stage/{{ .stage }}`, `path: {{ .overlay }}`.
- Sources in `env/` (Kustomize `base/` + `overlays/`). One overlay = one Argo CD app.
  - `base/`: NGINX Deployment + Service (`nginx`), no color, plus `default.conf` (ConfigMap `nginx-conf`, mounted over `/etc/nginx/conf.d/default.conf`): `ssi on` so `index.html` prints `nginx_version` and `hostname` (pod name), and `Cache-Control: no-store` so browsers never show a stale color. Verified with the real image on 2026-10-04.
  - `overlays/blue`, `overlays/green`: `nameSuffix`, `color` label (`includeSelectors: true`, which also sets the Service selector), `configMapGenerator` page from `index.html`, image tag in `images:`. Each renders `nginx-<color>` Deployment + Service.
  - `overlays/preview`: `nginx-preview` Service, pointing at the most recently deployed color. Same color as `nginx` after a `live` switch until the next `preview` promotion (like Argo Rollouts' preview Service). Own overlay/app because an app can be authorized for only one Stage.
  - `overlays/live`: `nginx` Service users hit; `spec.selector.color` decides traffic.
  - `preview` and `live` don't use `base/` (that would pull in a Deployment).
  - The `newTag` / `color` values on `main` are only the **starting state**, used when a branch has nothing rendered yet. Kargo sets them in its workspace copy of `main` before rendering.
- **State lives in the rendered branches**, read back on each promotion:
  - `preview`: clone `main` (`./src`), `stage/preview` (`./out`), `stage/live` (`./live`). Read the live color from `./live/live/service-nginx.yaml`. **Keep** the live color's rendered dir as is; **re-render** the idle color (new tag via `kustomize-set-image` on `./src`) and `preview/` (selector via `yaml-update` on `./src`). `delete` removes the old idle/preview dirs first. Commit + push `stage/preview`, `argocd-update` blue/green/preview with `desiredRevision: outputs.push.commit`.
  - Consequence: `main` changes (e.g. `base/`) reach the **idle color on the next release**, never the color serving users.
  - `live`: clone `main`, `stage/live` (`./out`), `stage/preview` (`./preview`). Read blue's image from `./preview/blue/deployment-nginx-blue.yaml`; if it equals `vars.image:<Freight tag>`, live = blue, else green. `yaml-update` `./src/.../live/service.yaml`, `git-clear` `./out`, render `live/`, commit + push `stage/live`, `argocd-update` `bluegreen-live`.
  - **First-run bootstrap** (missing rendered files): `yaml-parse` probes with `continueOnError: true`, then steps with `if: ${{ status('<probe>') == 'Errored' }}` render the starting state from `main`. If `stage/live` has no live Service yet, the `preview` Stage also commits + pushes the rendered `live/` to `stage/live` once, so `bluegreen-live` has a branch to track.
- One ApplicationSet `bluegreen` (`argocd/applicationset.yaml`, list generator of overlay + stage) generates four Applications, `bluegreen-<overlay>`. Edit the ApplicationSet, not the apps. `blue`/`green`/`preview` are authorized for Stage `preview`, `live` for Stage `live`. None auto-sync; Kargo's `argocd-update` syncs them.
- `desiredRevision` is safe only because each branch has a single writer. When both Stages pushed to `main`, the other Stage's commits moved the apps off the pinned revision, so Kargo marked the Stage Unhealthy while Argo CD showed Healthy. Don't let two Stages push to the same branch with `desiredRevision` set.
- `task status` / `task watch-status` read the cluster: each Service's selector color and that color's Deployment image tag.
- Taskfile runs `envsubst` restricted to the `.env` variables (`SUBST` var), so `${{ }}` Kargo expressions are never touched.

## Status

Tested on 2026-10-02 with the earlier edit-in-place design on `main`: `yaml-parse`, `kustomize-set-image`, Git credentials, both Stages, color alternation and `nginx-preview` all worked. **The rendered-branches design is untested.** New since then: three-way `git-clone` with `create: true`, probe + `if`/`status()` bootstrap, `delete`, `kustomize-build`, `git-clear`, `git-push` `targetBranch` with `outputs.push.commit`, `desiredRevision` restored. Rollback also untested.

If a promotion fails, check in this order:
1. **`yaml-parse` output references.** `outputs.<alias>.<name>` confirmed working. Now also reads rendered files: `spec.template.spec.containers[0].image` and `spec.selector.color`. If a path isn't found, check the rendered file names on the branch (Kargo's `kind-name.yaml` naming is assumed).
2. **`kustomize-set-image` config.** Uses `images[].image` + `tag`. If Kargo rejects `tag`, check the step's docs for the installed version.
3. **Ternary expressions** (`outputs.live.color == 'blue' ? 'green' : 'blue'`) must stay double-quoted in YAML. Unquoted, the ` : ` breaks parsing.
4. Git credentials: `secret.yaml` `repoURL` must match `GITOPS_REPO_URL` exactly, and the PAT needs write access.
5. `argocd-update` "not authorized": annotation must be `bluegreen-demo:<stage>` (set in the ApplicationSet template).
6. ApplicationSet not applied/generating: the ApplicationSet controller must be enabled on the Akuity Argo CD instance; if `akuity argocd apply` rejects the ApplicationSet kind, use `kubectl apply -n argocd` or `argocd appset create`.
7. **Bootstrap probes.** If the first `preview` promotion fails on the second `yaml-parse` (`live` or the active color), the `if: ${{ status('probe…') == 'Errored' }}` condition didn't match: check the probe step's actual status string in the Promotion details (might be `Failed`, not `Errored`).
8. **Apps show "unable to resolve stage/…"** before the first promotion: expected. The branches don't exist until the first `preview` promotion creates them.

## Known limitations

- `live` picks blue if blue's tag matches the Freight, otherwise green, without checking green. If the Freight's tag was since overwritten on both colors (two newer releases to `preview`), it switches to green anyway. If both colors run the same tag, it picks blue.
- `kubectl port-forward svc/...` pins to one pod and doesn't follow selector changes; restart it after each switch.
- `base/` changes (like the page version, added 2026-10-04) reach a color only when it's re-rendered: the idle color on each `preview` promotion of a new version. The live color keeps its old render until it's idle and gets a release, or until a fresh `task cleanup` + setup.
- To see what Kargo did, look at the `stage/preview` and `stage/live` branches, not `main`.
- The rendered Stage YAML repeats the idle-color ternary many times; step-level `vars` could tidy it but aren't used yet.
