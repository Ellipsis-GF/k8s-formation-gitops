# cert-manager + Cluster API via Argo CD (argocd-greg)

Pattern app-of-apps : une `Application` racine (`root-app.yaml`) surveille
`argocd/apps/`, qui declare quatre `Application` enfants.

```text
root (argocd-greg)
    |
    +-- cert-manager                 (wave 0, chart Jetstack, ns cert-manager)
    +-- cluster-api-operator         (wave 1, chart cluster-api-operator, ns capi-operator-system)
    +-- cluster-api-providers        (wave 2, argocd/providers/, ns capi-operator-system)
    |       |
    |       +-- CoreProvider cluster-api
    |       `-- InfrastructureProvider gcp (configSecret: capi-variables)
    |
    `-- cluster-api-workload-clusters (wave 3, argocd/clusters/)
            |
            +-- cluster-a (GKE manage, ns cluster-a, 1 MachinePool, 2 workers)
            `-- cluster-b (GKE manage, ns cluster-b, 1 MachinePool, 2 workers)
```

Les `sync-wave` garantissent l'ordre : cert-manager (dont cluster-api-operator
depend pour ses webhooks) est synchronise avant l'operateur, lui-meme avant
les CR Provider, eux-memes avant la creation des clusters enfants (qui ont
besoin du CoreProvider + InfrastructureProvider gcp deja `Ready`).

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

## Clusters enfants (cluster-a, cluster-b)

Deux clusters GKE manages (`GCPManagedCluster` + `GCPManagedControlPlane` +
`MachinePool`/`GCPManagedMachinePool`), projet `k8s-formation`, region
`europe-west9`, `releaseChannel: regular` (pas de version epinglee), 1
node pool de 2 workers (`e2-medium`) chacun. Le support GKE et MachinePool
sont deja actives dans le Secret `capi-variables`
(`EXP_CAPG_GKE`/`EXP_MACHINE_POOL`).

Suivre la progression :

```sh
kubectl get clusters -A
kubectl get gcpmanagedcontrolplanes -A
kubectl get machinepools -A
```

Le provisioning GKE reel (control plane + node pool) prend plusieurs
minutes une fois les CR synchronisees par Argo CD.

## Bootstrap

```sh
kubectl apply -n argocd-greg -f argocd/root-app.yaml
kubectl get applications -n argocd-greg
```

Le reste (cert-manager, cluster-api-operator, providers) est ensuite gere
automatiquement par Argo CD (`prune: true`, `selfHeal: true`).
