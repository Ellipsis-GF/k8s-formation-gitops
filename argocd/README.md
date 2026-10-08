# cert-manager + Cluster API via Argo CD (argocd-greg)

Pattern app-of-apps : une `Application` racine (`root-app.yaml`) surveille
`argocd/apps/`, qui declare les `Application` enfants.

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
    +-- cluster-api-workload-clusters (wave 3, argocd/clusters/)
            |
            +-- cluster-a (GKE manage, ns cluster-a, 1 MachinePool, 1 worker)
            `-- cluster-b (GKE manage, ns cluster-b, 1 MachinePool, 1 worker)
    +-- cluster-registration         (wave 2, labels des Secrets Argo CD pour A et B)
    +-- openmetadata-dependencies    (wave 5, cluster-a, MySQL + OpenSearch)
    `-- openmetadata                 (wave 6, cluster-a, serveur + Jobs Kubernetes)
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
| OpenMetadata dependencies | `apps/openmetadata-dependencies.yaml`             | 2.0.2     |
| OpenMetadata              | `apps/openmetadata.yaml`                          | 2.0.2     |

`repoURL` pointe sur `git@github.com:Ellipsis-GF/k8s-formation-gitops.git`
(SSH), pour matcher le Secret `repo-capi-gitops` (label
`argocd.argoproj.io/secret-type=repository`) deja cree dans `argocd-greg`.

## Clusters enfants (cluster-a, cluster-b)

Deux clusters GKE manages (`GCPManagedCluster` + `GCPManagedControlPlane` +
`MachinePool`/`GCPManagedMachinePool`), projet `k8s-formation`, region
`europe-west9`, `releaseChannel: regular` (pas de version epinglee), 1
node pool de 1 worker (`e2-medium`) chacun. Le support GKE et MachinePool
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

## OpenMetadata sur cluster-a

Les Applications `openmetadata-dependencies` et `openmetadata` ciblent le
cluster Argo CD nomme `cluster-a`, dans le namespace `openmetadata`.
L'Application `cluster-registration` maintient les Secrets `argocd-cluster-a`
et `argocd-cluster-b` dans `argocd-greg`, avec leur nom de cluster et le label
`argocd.argoproj.io/secret-type: cluster`. Elle les synchronise automatiquement
et retablit le label s'il est retire.

Les donnees d'authentification (`data.config`) doivent etre provisionnees hors
Git avant le premier enregistrement. Les manifests seuls ne generent pas de
credentials et ne suffisent donc pas a connecter un cluster neuf a Argo CD.
CAPG actualise l'endpoint DNS (`data.server`) via `argoCDClusterSecretRef`.
Ces deux champs sont ignores au diff et preserves au sync ; la migration du
`kubectl apply` initial est desactivee pour conserver ses champs existants.
Les Secrets sont conserves lors d'un prune ou de la suppression de l'Application.

La configuration est volontairement reduite pour le cluster de formation :

- MySQL et OpenSearch sont deployes avec un volume persistant de 10 Gi chacun ;
- Airflow et Fuseki sont desactives ;
- les pipelines d'ingestion utilisent des Jobs Kubernetes ;
- le service OpenMetadata reste de type `ClusterIP` ;
- les mots de passe MySQL declares dans Git sont reserves a la demonstration.

Verifier le deploiement :

```sh
kubectl get applications -n argocd-greg
kubectl --context cluster-a get pods,pvc -n openmetadata
```

Acceder a l'interface depuis le poste local :

```sh
kubectl --context cluster-a -n openmetadata port-forward service/openmetadata 8585:8585
```

Ouvrir ensuite `http://localhost:8585`.

## Bootstrap

```sh
kubectl apply -n argocd-greg -f argocd/root-app.yaml
kubectl get applications -n argocd-greg
```

Le reste (cert-manager, cluster-api-operator, providers) est ensuite gere
automatiquement par Argo CD (`prune: true`, `selfHeal: true`).
