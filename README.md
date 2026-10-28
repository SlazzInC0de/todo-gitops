# todo-gitops

GitOps config repo for the todo app. **Argo CD watches this repo and makes the
cluster match it.** Nobody runs `kubectl apply` against the app after setup.

```
apps/todo/
├── kustomization.yaml   # namespace + the deployed image tag (CI edits this)
├── configmap.yaml       # non-secret configuration
├── postgres.yaml        # StatefulSet + headless Service + PVC
├── deployment.yaml      # todo-api: 2 replicas, probes, limits, hardened
├── service.yaml         # NodePort 30080 -> http://localhost:8080 on kind
├── servicemonitor.yaml  # Prometheus scrape config
└── dashboard.yaml       # Grafana dashboard as code
argocd/
└── todo-app.yaml        # the Argo CD Application pointing at apps/todo
```

## How changes arrive

1. A PR is merged in the **app repo**.
2. CI builds `ghcr.io/<user>/todo-api:<sha>` and commits that tag to
   `apps/todo/kustomization.yaml` here (`deploy: todo-api <sha>`).
3. Argo CD notices the new commit (within ~60 s) and rolls it out.

## Common operations

| Task | How |
| --- | --- |
| See what is deployed | `newTag` in `apps/todo/kustomization.yaml` |
| Roll back | `git revert <the deploy commit>` and push |
| Change config | Edit `configmap.yaml`, open a PR, merge |
| Scale | Change `replicas` in `deployment.yaml` (not `kubectl scale`: Argo CD reverts it) |
| Preview rendered YAML | `kubectl kustomize apps/todo` |

## Secrets

The database Secret `todo-db` is **not** in Git. It is created once per
cluster with `make secret` from the app repo. Production teams use
Sealed Secrets or External Secrets Operator to manage secrets through Git safely.
