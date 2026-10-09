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

Les versions ci-dessous sont celles declarees dans les manifests :

| Composant              | Fichier                                         | Version   |
|-------------------------|--------------------------------------------------|-----------|
| cert-manager            | `apps/cert-manager.yaml`                         | v1.16.3   |
| cluster-api-operator    | `apps/cluster-api-operator.yaml`                 | 0.16.0    |
| CoreProvider (CAPI)     | `providers/core-provider.yaml`                   | v1.9.5    |
| InfrastructureProvider gcp (CAPG perso) | `providers/infrastructure-provider-gcp.yaml` | v1.8.2-dns.2 |
| OpenMetadata dependencies | `apps/openmetadata-dependencies.yaml`             | 2.0.2     |
| OpenMetadata              | `apps/openmetadata.yaml`                          | 2.0.2     |

`repoURL` pointe sur `git@github.com:Ellipsis-GF/k8s-formation-gitops.git`
(SSH), pour matcher le Secret `repo-capi-gitops` (label
`argocd.argoproj.io/secret-type=repository`) deja cree dans `argocd-greg`.

## Clusters enfants (cluster-a, cluster-b)

Deux clusters GKE manages (`GCPManagedCluster` + `GCPManagedControlPlane` +
`MachinePool`/`GCPManagedMachinePool`), projet `k8s-formation`, region
`europe-west9`, `releaseChannel: regular` (pas de version epinglee), 1
node pool de 1 worker (`e2-standard-4`) chacun. Le support GKE et MachinePool
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

## Reseau des clusters enfants

Les deux clusters conservent le VPC `default`, avec un subnet dedie par cluster :

| Cluster | Region du subnet | Subnet | Plage primaire des nodes |
|---------|------------------|--------|--------------------------|
| A | europe-west9 | cluster-a-subnet | 10.20.0.0/24 |
| B | europe-west9 | cluster-b-subnet | 10.30.0.0/24 |

Ces subnets sont declares dans `clusters/cluster-a/gcp-managed-cluster.yaml`
et son equivalent pour B, sous `spec.network.subnets`, avec
`privateGoogleAccess: true`. Le workflow retenu consiste a creer les subnets
manuellement avant la synchronisation Argo CD. CAPG reutilise les subnets existants
et transmet a GKE le premier subnet declare dont la region correspond a celle
du cluster. Declarer un seul subnet par cluster pour rendre cette selection explicite.
CAPG peut creer un subnet absent : verifier leur existence avant le premier sync.

Le CAPG personnalise est conserve, sans modification du controleur ou des CRD.
Les control planes utilisent `clusterNetwork.useIPAliases: true` : GKE alloue
automatiquement la plage secondaire Pods. Les Services peuvent utiliser la plage
geree par GKE `34.118.224.0/20`, sans plage secondaire Services sur le subnet.
Cette plage est utilisee a titre prive et n'est pas annoncee sur Internet ;
les ClusterIP restent propres a chaque cluster. Les CIDR Pods precedemment declares
en `192.168.0.0/16` ont ete retires des objets `Cluster` et
`GCPManagedControlPlane`, car cette version ne les transmet pas a GKE.

Changer ces manifests ne deplace pas un cluster existant vers le nouveau subnet.
Il faut preparer le remplacement ou la recreation des clusters, avec sauvegarde
et restauration des workloads, Secrets et volumes, avant la bascule Argo CD/OCM.
Le subnet `default` existant reste conserve. Un subnet cree par CAPG peut etre
supprime lors de la suppression de son cluster : tenir compte de ce cycle de vie.
Pour les subnets manuels, utiliser une description distincte du marqueur de
propriete CAPG et ne pas renseigner `spec.network.subnets[].description`.
Le controleur personnalise utilise la description pour determiner les subnets
a supprimer lors de la suppression du cluster.

Verifier le reseau reel apres provisioning (la valeur `subnetwork` doit etre
`cluster-a-subnet` ou `cluster-b-subnet`) :

```sh
gcloud container clusters describe cluster-a --project=k8s-formation \
  --location=europe-west9-a --format='yaml(network,subnetwork,ipAllocationPolicy)'
gcloud container clusters describe cluster-b --project=k8s-formation \
  --location=europe-west9-a --format='yaml(network,subnetwork,ipAllocationPolicy)'
gcloud compute networks subnets describe cluster-a-subnet --project=k8s-formation \
  --region=europe-west9 --format='yaml(network,ipCidrRange,privateIpGoogleAccess,secondaryIpRanges)'
gcloud compute networks subnets describe cluster-b-subnet --project=k8s-formation \
  --region=europe-west9 --format='yaml(network,ipCidrRange,privateIpGoogleAccess,secondaryIpRanges)'
```

## Verification de Private Service Connect et du DNS GKE

Les manifests CAPG activent l'acces DNS et desactivent l'acces IP au control plane
(`dnsAccess: true`, `ipAccess: false`). Ils ne creent ni endpoint PSC Google APIs
ni zone Cloud DNS. Ces ressources existent hors de ce depot.

L'inventaire du 8 octobre 2026 montrait les endpoints globaux suivants ; ces
valeurs sont une reference a reverifier, pas une validation de l'etat actuel :

| VPC source du client | Endpoint PSC | IP | Cible |
|---------------------|--------------|----|-------|
| k8s-lab-7cse (Management) | pscgkemgmt | 10.100.100.10 | all-apis |
| default (A/B) | pscgkechildren | 10.100.100.11 | all-apis |

Un endpoint PSC Google APIs s'utilise depuis son VPC ; le peering ne le rend pas
accessible depuis l'autre VPC. Les nouveaux subnets A/B restent dans `default`
et peuvent donc utiliser `pscgkechildren`, avec Private Google Access active.
La reponse DNS depend du VPC dans lequel tourne le client. Les endpoints PSC
Google APIs utilisent des adresses internes globales hors des plages des subnets,
y compris celles des VPC peers. Ne pas creer un subnet englobant
`10.100.100.10` ou `10.100.100.11`. Les PSC et les zones DNS sont geres manuellement
et restent utilisables apres recreation de A/B dans le meme VPC.

### 1. Controler les endpoints, zones et FQDN actuels

Si gcloud indique une erreur de reauthentification, executer d'abord
`gcloud auth login` sur le poste local. Les commandes suivantes sont en lecture seule :

```sh
gcloud compute forwarding-rules describe pscgkemgmt --global \
  --project=k8s-formation --format='yaml(name,IPAddress,network,target)'
gcloud compute forwarding-rules describe pscgkechildren --global \
  --project=k8s-formation --format='yaml(name,IPAddress,network,target)'
gcloud dns managed-zones list --project=k8s-formation \
  --filter='visibility=private' \
  --format='yaml(name,dnsName,privateVisibilityConfig)'
gcloud dns record-sets list --project=k8s-formation --zone=gke-goog-private
gcloud dns record-sets list --project=k8s-formation --zone=gke-hub-ocm-private
gcloud container clusters list --project=k8s-formation \
  --format='yaml(name,location,controlPlaneEndpointsConfig)'
```

Attendu : chaque PSC est dans le VPC indique ci-dessus, avec une cible `all-apis`.
Les clusters A/B doivent avoir un endpoint DNS renseigne et
`ipEndpointsConfig.enabled: false`. Copier les FQDN actuels dans les tests suivants ;
ils peuvent changer apres recreation.

Le 8 octobre, la zone `gke-goog-private` (`gke.goog.`), visible depuis Management,
contenait un A `gke.goog.` vers `10.100.100.10` et un CNAME `*.gke.goog.` vers
`gke.goog.`. La zone `gke-hub-ocm-private`, visible depuis `default`, contenait
un A pour le FQDN exact du Management vers `10.100.100.11`. Ce dernier record doit
correspondre au FQDN actuel du Management. Cette zone specifique ne couvre pas
automatiquement les endpoints A/B.

### 2. Tester DNS et HTTPS depuis le VPC du client

Utiliser un pod existant disposant de `nslookup` et `curl`, sans creer de pod de test.
Le contexte kubectl doit cibler le cluster source ; remplacer le namespace et le
nom de pod ci-dessous. Si ces outils ne sont pas installes, utiliser un autre pod
ou une VM existante dans le meme VPC.

```sh
GKE_DNS='REMPLACER_PAR_LE_FQDN_ACTUEL_SANS_HTTPS'
NETCHECK_NS='REMPLACER_PAR_LE_NAMESPACE'
NETCHECK_POD='REMPLACER_PAR_LE_POD_EXISTANT'
kubectl -n "$NETCHECK_NS" exec "$NETCHECK_POD" -- nslookup "$GKE_DNS"
kubectl -n "$NETCHECK_NS" exec "$NETCHECK_POD" -- \
  curl --connect-timeout 5 --max-time 15 -sS -o /dev/null \
  -w 'ip=%{remote_ip} http=%{http_code}\n' "https://${GKE_DNS}/version"
```

Depuis Management vers les API A/B, attendre `10.100.100.10`. Depuis A ou B vers
l'API Management, attendre `10.100.100.11`. Tester les deux directions et les deux
clusters enfants. Une IP publique indique que cet appel ne passe pas par le PSC.
Un test depuis le poste local ne valide pas la resolution DNS des pods.

Avec verification TLS active, une reponse HTTP 401/403 prouve que la connexion
HTTPS aboutit, mais pas que les credentials IAM/RBAC sont valides. Un timeout ou
`http=000` indique un probleme de connexion/TLS a investiguer. Si necessaire,
isoler la connectivite PSC du DNS avec `curl --resolve` en conservant le hostname TLS :

```sh
PSC_IP='10.100.100.10' # utiliser 10.100.100.11 pour un client dans default
kubectl -n "$NETCHECK_NS" exec "$NETCHECK_POD" -- \
  curl --connect-timeout 5 --max-time 15 -sS -o /dev/null \
  --resolve "${GKE_DNS}:443:${PSC_IP}" \
  -w 'ip=%{remote_ip} http=%{http_code}\n' "https://${GKE_DNS}/version"
```

Si le test force fonctionne mais que le test normal echoue, examiner les zones
DNS et leur visibilite. Si les deux echouent, examiner PSC, egress firewall,
NetworkPolicies et Private Google Access. Ne pas utiliser `curl -k`, qui masque
les erreurs de certificat.

### 3. Verifier les consommateurs authentifies

```sh
kubectl -n argocd-greg get applications
kubectl -n argocd-khalil get applications
kubectl get clusters.cluster.x-k8s.io -A
kubectl get gcpmanagedcontrolplanes -A
kubectl get managedclusters.cluster.open-cluster-management.io
```

Comparer les endpoints publies par CAPG et CAPI aux FQDN GKE actuels. Verifier les
logs des consommateurs en cas d'erreur de connexion et la condition OCM
`ManagedClusterConditionAvailable=True`. Pour les registrations Argo CD, lire
uniquement `data.server` (ne pas afficher les credentials `data.config`) :

```sh
for argo_ns in argocd-greg argocd-khalil; do
  for child_name in cluster-a cluster-b; do
    printf '%s/%s: ' "$argo_ns" "$child_name"
    kubectl -n "$argo_ns" get secret "argocd-${child_name}" \
      -o jsonpath='{.data.server}' | base64 --decode
    printf '\n'
  done
done
```

Le statut des Applications et un acces kubectl depuis le poste local ne prouvent
pas a eux seuls l'utilisation du PSC : combiner la resolution privee, l'IP distante
du test HTTPS et la validation des consommateurs authentifies.

References : [acces DNS GKE via PSC](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/latest/network-isolation#dns-based_endpoint),
[configuration des endpoints PSC Google APIs](https://docs.cloud.google.com/vpc/docs/configure-private-service-connect-apis).

### Controle effectue le 9 octobre 2026 (Europe/Paris)

Cet etat a ete observe avant la suppression du cluster A.

Les endpoints PSC, les cibles `all-apis` et les vues DNS decrits ci-dessus ont ete
reverifies en lecture seule. Les tests HTTPS ont conserve la verification TLS et
n'ont transmis aucun credential : les reponses 403 valident la connectivite,
pas les droits IAM/RBAC des consommateurs.

| Trajet | Pod utilise pour HTTPS | IP distante | HTTP |
|--------|------------------------|-------------|------|
| Management vers A | monitoring/grafana | 10.100.100.10 | 403 |
| Management vers B | monitoring/grafana | 10.100.100.10 | 403 |
| A vers Management | openmetadata/opensearch-0 | 10.100.100.11 | 403 |
| B vers Management | gitlab/gitlab-0 | 10.100.100.11 | 403 |

La resolution via `getent hosts` a aussi ete verifiee depuis les deux controleurs
Argo CD vers A/B (10.100.100.10), et depuis A/B vers Management (10.100.100.11).
Les images des controleurs Argo CD ne disposent pas de `curl` ; les tests HTTPS
ont utilise les pods applicatifs existants ci-dessus, sans creer de ressource.
Les quatre Secrets de registration Argo CD referencent les FQDN GKE actuels.
Les Applications OpenMetadata et plusieurs Applications deployees sur A/B sont
`Synced`/`Healthy`, sans erreur de connexion declaree.

Lors de ce controle, A/B utilisaient encore le subnet `default`
(`10.200.0.0/20`) et les subnets dedies n'etaient pas encore crees.
Une verification ulterieure a confirme la creation des subnets `cluster-a-subnet`
et `cluster-b-subnet`, l'absence du cluster GKE A et la presence de B sur `default`.
La recreation de A etait bloquee par l'absence de `spec.network.subnets` dans
l'objet deploye : les modifications locales doivent etre publiees puis synchronisees.
Les objets CAPI
`Cluster.spec.controlPlaneEndpoint` portent d'anciens FQDN, contrairement aux
`GCPManagedControlPlane` et aux registrations Argo CD. OCM indique encore
`ManagedClusterConditionAvailable=Unknown` pour A/B, avec arret des mises a jour
des leases. Le fonctionnement PSC confirme ci-dessus ne suffit donc pas a
valider le fonctionnement de CAPI et d'OCM ; ces ecarts doivent etre examines
separement avant la migration.

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
CAPG actualise l'endpoint DNS (`data.server`) via `argoCDClusterSecretRefs`,
pour les registrations declarees dans `argocd-greg` et `argocd-khalil`.
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
