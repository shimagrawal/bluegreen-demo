# CLAUDE.md

Kubernetes-native blue-green demo driven by Kargo + Argo CD on the Akuity Platform. Companion to the `akuity-workshop` repo (`../akuity-workshop`), which uses the same `.env` + `envsubst` + Taskfile conventions.

## Working rules

- Don't commit or push without asking first. The user pushes manually.
- Never commit `.env` (it holds a GitHub PAT).

## Design

- Kustomize `env/base/` + `env/overlays/`, no other folders. One overlay = one Argo CD app path.
  - `base/`: NGINX Deployment + Service (`nginx`), no color.
  - `overlays/blue`, `overlays/green`: `nameSuffix`, `color` label (`includeSelectors: true`, which also sets the Service selector), `configMapGenerator` page from `index.html`, image tag in `images:`. Each renders `nginx-<color>` Deployment + Service; the per-color Service is how you test the idle color.
  - `overlays/live`: the `nginx` Service users hit. Its `spec.selector.color` decides which color gets traffic. Blue is live initially. Doesn't use `base/` (that would pull in a Deployment); it holds its own `service.yaml`.
- Kargo Project `bluegreen-demo`, Stages `preview → live` (`kargo/stages.yaml`). Steps are inline in each Stage, no PromotionTask.
  - `preview`: `yaml-parse` the live color from `overlays/live/service.yaml`, `kustomize-set-image` in the *other* color's overlay, commit + push to `main`, `argocd-update` `bluegreen-blue` + `bluegreen-green` (syncing the unchanged color is a no-op).
  - `live`: `yaml-parse` `images[0].newTag` from `overlays/blue/kustomization.yaml`; if it equals the Freight's tag switch to blue, else green. `yaml-update` `overlays/live/service.yaml`, commit + push, `argocd-update` `bluegreen-live`.
- One ApplicationSet `bluegreen` (`argocd/applicationset.yaml`, list generator of overlay + stage) generates three Applications, `bluegreen-<overlay>`. Edit the ApplicationSet, not the apps. `blue`/`green` are authorized for Stage `preview`, `live` for Stage `live`. None auto-sync; Kargo's `argocd-update` syncs them.
- Kargo edits files directly on `main` (no rendered branches). Both Stages push to the same branch, so `argocd-update` must **not** set `desiredRevision`: the other Stage's next commit moves the apps off that revision and Kargo marks the Stage Unhealthy even though Argo CD is Healthy.
- Taskfile runs `envsubst` restricted to the `.env` variables (`SUBST` var), so `${{ }}` Kargo expressions are never touched.

## Status

Tested on 2026-10-02: `preview` (1.31.6 → green) and `live` (switch to green) both promoted and worked end to end, so the `yaml-parse` / `kustomize-set-image` syntax and Git credentials are good. Not yet tested: a second release onto blue, rollback, and the `desiredRevision` fix below.

If a promotion fails, check in this order:
1. **`yaml-parse` output references.** Stages use `${{ outputs.live.color }}` / `${{ outputs.blue.tag }}`, and `fromExpression: images[0].newTag` for the kustomization. Confirmed working on 2026-10-02.
2. **`kustomize-set-image` config.** Uses `images[].image` + `tag`. If Kargo rejects `tag`, check the step's docs for the installed version.
3. **Ternary expressions** (`outputs.live.color == 'blue' ? 'green' : 'blue'`) must stay double-quoted in YAML. Unquoted, the ` : ` breaks parsing.
4. Git credentials: `secret.yaml` `repoURL` must match `GITOPS_REPO_URL` exactly, and the PAT needs write access.
5. `argocd-update` "not authorized": annotation must be `bluegreen-demo:<stage>` (set in the ApplicationSet template).
6. ApplicationSet not applied/generating: the ApplicationSet controller must be enabled on the Akuity Argo CD instance; if `akuity argocd apply` rejects the ApplicationSet kind, use `kubectl apply -n argocd` or `argocd appset create`.

## Known limitations

- `live` picks blue if blue's tag matches the Freight, otherwise green, without checking green. If the Freight's tag was since overwritten on both colors (two newer releases to `preview`), it switches to green anyway. If both colors run the same tag, it picks blue.
- `kubectl port-forward svc/...` pins to one pod and doesn't follow selector changes; restart it after each switch.
- Kargo pushes to `main`, so `git pull` before local edits.
