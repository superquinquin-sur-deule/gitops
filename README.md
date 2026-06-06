## SQQ GitOps

Dépôt **GitOps** du cluster kubernetes de la coop. Tout l'état du cluster est décrit
ici en YAML : [Flux](https://fluxcd.io/) lit ce dépôt et réconcilie le cluster pour
qu'il corresponde à ce qui est versionné. Pour modifier le cluster, on modifie le
dépôt — on ne `kubectl apply` rien à la main.

> Le guide d'installation et de bootstrap complet est dans **[docs/SETUP.md](./docs/SETUP.md)**.

## La stack

| Couche           | Outil                                                                                                                                                                                                                                                                                     |
|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| OS / nœuds       | [Talos Linux](https://www.talos.dev/) (config via [talhelper](https://github.com/budimanjojo/talhelper), dossier `talos/`)                                                                                                                                                                |
| GitOps           | [Flux](https://fluxcd.io/) (operator + instance)                                                                                                                                                                                                                                          |
| Réseau           | [Cilium](https://cilium.io/) (CNI), [Envoy Gateway](https://gateway.envoyproxy.io/), [cloudflared](https://github.com/cloudflare/cloudflared) (tunnel), [external-dns](https://github.com/kubernetes-sigs/external-dns) + [k8s-gateway](https://github.com/k8s-gateway/k8s_gateway) (DNS) |
| Secrets          | [OpenBao](https://openbao.org/) + [External Secrets](https://external-secrets.io/) ; [SOPS](https://github.com/getsops/sops)/age pour les secrets versionnés (bootstrap)                                                                                                                  |
| Auth             | [Keycloak](https://www.keycloak.org/) (OIDC)                                                                                                                                                                                                                                              |
| Stockage         | [Rook-Ceph](https://rook.io/), [OpenEBS](https://openebs.io/) ; sauvegardes via [VolSync](https://volsync.readthedocs.io/) + [Kopia](https://kopia.io/)                                                                                                                                   |
| Bases de données | [Crunchy PGO](https://www.crunchydata.com/) (PostgreSQL), MongoDB                                                                                                                                                                                                                         |
| Observabilité    | kube-prometheus-stack, [Grafana](https://grafana.com/), [Loki](https://grafana.com/oss/loki/) + Vector, blackbox/smartctl exporters                                                                                                                                                       |

## Structure du dépôt

```
kubernetes/
├── flux/cluster/      # Point d'entrée Flux : la Kustomization racine qui pointe sur apps/
├── apps/              # Toutes les applications, une par namespace
│   ├── auth/          # Keycloak
│   ├── database/      # crunchy-pgo, mongodb
│   ├── default/       # apps métier : outline, rocketchat, accueil, sesame, scannettes...
│   ├── network/       # cloudflare-tunnel, envoy-gateway, external-dns, k8s-gateway
│   ├── observability/ # prometheus, grafana, loki, vector, exporters
│   ├── openbao/       # gestion des secrets
│   ├── kube-system/   # cilium, coredns, spegel, reloader...
│   └── ...            # external-secrets, rook-ceph, openebs, volsync, routines
└── components/        # briques réutilisables (sops, volsync) montées par les apps

talos/        # configuration des nœuds Talos (talconfig.yaml + clusterconfig généré)
bootstrap/    # ressources de bootstrap (clés SOPS/deploy, helmfile initial)
template/     # templating makejinja (rendu depuis cluster.toml)
```

### Anatomie d'une app

Chaque application suit le même schéma (exemple `apps/default/outline/`) :

```
ks.yaml               # Flux Kustomization : dépendances, namespace cible, chemin de l'app
app/
├── kustomization.yaml
├── helmrelease.yaml      # ou ocirepository.yaml + manifests
├── externalsecret.yaml   # secrets tirés d'OpenBao via External Secrets
└── keycloakclient.yaml   # client OIDC si l'app utilise l'auth
```

Le `ks.yaml` déclare les `dependsOn` (ex : attendre `pg-main` et les stores External
Secrets avant de déployer), et injecte les valeurs partagées depuis le secret
`cluster-secrets`.

## Workflow quotidien

1. **Modifier** les manifests dans `kubernetes/`.
2. **Valider en local** sans pousser (reproduit la CI) :
   ```sh
   uvx flux-local test --enable-helm --all-namespaces --path kubernetes/flux/cluster
   ```
3. **Commit + push** sur `main`.
4. **Flux réconcilie** automatiquement (webhook GitHub, sinon poll toutes les heures).
   Pour forcer : `just kube reconcile`.

Les dépendances (charts Helm, images, GitHub Actions) sont mises à jour
automatiquement par [Renovate](https://www.mend.io/renovate) qui ouvre des PR.

## Secrets

- Les secrets applicatifs vivent dans **OpenBao** et sont injectés dans le cluster
  via **External Secrets** (`externalsecret.yaml`).
- Quelques secrets nécessaire lors du bootstrap sont versionnés chiffrés avec **SOPS/age**
  (fichiers `*.sops.yaml`) — la clé est dans `age.key` (non versionnée).

## Commandes utiles

```sh
just                      # liste toutes les recettes disponibles
just configure            # re-rend la config depuis cluster.toml (makejinja)
just kube reconcile       # force Flux à se synchroniser sur le dépôt
just talos generate-config
just talos apply-node <ip>
flux get ks -A            # état des Kustomizations
flux get hr -A            # état des HelmReleases
```

L'environnement (CLI : kubectl, flux, talosctl, sops, openbao…) est géré par
[mise](https://mise.jdx.dev/) — voir `.mise.toml`. `KUBECONFIG`, `TALOSCONFIG` et
les clés SOPS sont configurés automatiquement à l'activation.
