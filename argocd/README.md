# cert-manager + Cluster API via Argo CD (argocd-greg)

Pattern app-of-apps : une `Application` racine (`root-app.yaml`) surveille
`argocd/apps/`, qui declare trois `Application` enfants.

```text
root (argocd-greg)
    |
    +-- cert-manager            (wave 0, chart Jetstack, ns cert-manager)
    +-- cluster-api-operator    (wave 1, chart cluster-api-operator, ns capi-operator-system)
    `-- cluster-api-providers   (wave 2, argocd/providers/, ns capi-operator-system)
            |
            +-- CoreProvider cluster-api
            `-- InfrastructureProvider gcp (configSecret: capi-variables)
```

Les `sync-wave` garantissent l'ordre : cert-manager (dont cluster-api-operator
depend pour ses webhooks) est synchronise avant l'operateur, lui-meme avant
les CR Provider.

## Prerequis (deja en place selon toi)

- Namespace `argocd-greg` avec l'instance Argo CD (cf. `capabilities/gitops/argocd`).
- Namespace `capi-operator-system` avec le Secret `capi-variables` contenant
  les credentials GCP attendus par CAPG (`GCP_B64ENCODED_CREDENTIALS`, etc.).
- `ClusterRoleBinding/argocd-greg-cluster-admin` (dans `capabilities/`) pour
  que le controller Argo CD puisse installer CRDs/RBAC cluster-wide.

## A verifier avant le premier sync

Les versions ci-dessous sont des valeurs recentes mises par defaut, a
confirmer/ajuster selon ce qui est deja valide dans ton environnement :

| Composant              | Fichier                                         | Version   |
|-------------------------|--------------------------------------------------|-----------|
| cert-manager            | `apps/cert-manager.yaml`                         | v1.16.3   |
| cluster-api-operator    | `apps/cluster-api-operator.yaml`                 | 0.16.0    |
| CoreProvider (CAPI)     | `providers/core-provider.yaml`                   | v1.9.5    |
| InfrastructureProvider gcp (CAPG) | `providers/infrastructure-provider-gcp.yaml` | v1.8.1    |

`repoURL` pointe sur `git@github.com:Ellipsis-GF/k8s-formation-gitops.git`
(SSH), pour matcher le Secret `repo-capi-gitops` (label
`argocd.argoproj.io/secret-type=repository`) deja cree dans `argocd-greg`.

## Bootstrap

```sh
kubectl apply -n argocd-greg -f argocd/root-app.yaml
kubectl get applications -n argocd-greg
```

Le reste (cert-manager, cluster-api-operator, providers) est ensuite gere
automatiquement par Argo CD (`prune: true`, `selfHeal: true`).
