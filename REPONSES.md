# TP20 : réponses aux questions

## Partie 1 : bootstrap

* `flux bootstrap github` crée le dépôt `tp20-flux` (s'il n'existe pas), y **commite ses propres manifests** sous `clusters/lab/flux-system/`, installe les contrôleurs dans le cluster, puis configure Flux pour **se synchroniser depuis ce même dossier**. Flux se gère lui-même en GitOps : pour le mettre à jour, on relance le bootstrap ou on modifie Git.
* `gotk-components.yaml` : les CRD et les Deployments des contrôleurs (source, kustomize, helm, notification).
* `gotk-sync.yaml` : un **GitRepository** `flux-system` (URL du dépôt, branche `main`, intervalle) et une **Kustomization** `flux-system` qui applique `./clusters/lab`. Tout fichier ajouté dans `clusters/lab/` (comme `apps.yaml`) est donc déployé.
* **Clé de déploiement** (*Settings → Deploy keys*) : le bootstrap génère une paire de clés SSH. La clé privée reste dans le Secret `flux-system` du cluster, la clé publique est enregistrée sur le dépôt. Le cluster lit **uniquement ce dépôt**, en **lecture seule** par défaut (ou en écriture avec `--read-write-key` pour l'image automation). Le jeton personnel ne sert qu'au bootstrap et n'est pas stocké dans le cluster.

| Contrôleur | Rôle |
|---|---|
| source-controller | récupère les sources (GitRepository, HelmRepository, OCIRepository, Bucket) et les met à disposition comme artefacts |
| kustomize-controller | applique les Kustomizations Flux (rendu Kustomize, apply côté serveur, prune, santé) |
| helm-controller | gère les HelmRelease (install, upgrade, test, rollback) avec le SDK Helm |
| notification-controller | envoie les événements (Slack, Teams, webhook…) et reçoit les webhooks (déclenchement immédiat) |

## Partie 2 : application Kustomize

* `apps.yaml` : Kustomization `apps-dev` → `./apps/dev`, `interval: 1m`, **`prune: true`** (suppression des ressources retirées de Git), **`wait: true`** (n'est *Ready* que si les ressources sont en bonne santé), **`timeout: 2m`**.
* **Dérive** (`scale --replicas=4`) : rien d'immédiat. À la **réconciliation suivante** (au plus 1 minute), Flux réapplique Git et le Deployment revient à 2 replicas.
  * **Argo CD** (self-heal) surveille les ressources en continu (watch) et corrige en **quelques secondes**, en affichant l'état *OutOfSync* dans l'UI.
  * **Flux** corrige **périodiquement** (`interval`) : la dérive peut durer jusqu'à un intervalle, sans état visuel (seulement `flux get` et les événements).
* **Erreur** (`ressources:` au lieu de `resources:`) : `flux get kustomizations` affiche `apps-dev False` avec `kustomize build failed … unknown field "ressources"`. `flux logs` et `kubectl describe` donnent le même message. Les ressources déjà déployées **ne sont pas touchées** : l'erreur bloque seulement les nouvelles versions. Correction par `git revert`.

## Partie 3 : suspendre, reprendre, supprimer

* `flux suspend` arrête la réconciliation (maintenance, débogage) : la modification manuelle (3 replicas) est **conservée**.
* `flux resume` relance la réconciliation : **Git reprend la main** et le Deployment revient à 2.
* **Prune** : le Service retiré de Git est supprimé du cluster, puis recréé après le `git revert`.

## Partie 4 : HelmRelease

* Flux utilise le **SDK Helm** : c'est une vraie release Helm, visible avec `helm list` et `helm history` (contrairement à Argo CD, qui fait seulement `helm template`).
* Le chart vient du même dépôt Git (`chart: ./charts/whoami`, source `GitRepository flux-system`) ; les `values` de la HelmRelease surchargent `values.yaml`.
* **Remédiation** : avec `image.tag: v9.9.9`, les Pods ne sont jamais prêts, et la mise à jour échoue après `timeout: 1m`. Flux réessaie 2 fois (`retries: 2`) puis fait un **rollback** (`strategy: rollback`, `remediateLastFailure: true`). `helm history` montre les révisions `failed` puis `Rollback to <révision saine>`. C'est l'équivalent automatique de `helm upgrade --atomic` (TP15). La HelmRelease reste en échec tant que Git n'est pas corrigé (`git revert`).

## Partie 5 : comparaison Argo CD / Flux

| Critère | Argo CD | Flux |
|---|---|---|
| Installation et mise à jour | un manifest (`kubectl apply`) ou un chart ; mises à jour à la main ou en se gérant lui-même par une Application | `flux bootstrap` : Flux écrit ses manifests dans Git et **se gère lui-même** en GitOps ; mise à jour = relancer le bootstrap |
| Interface (UI / CLI / CRD) | **UI web riche** (arbre, diff, logs, sync manuel) + CLI `argocd` + CRD `Application` / `ApplicationSet` | **pas d'UI** native (UIs tierces : Weave GitOps, Headlamp…) ; CLI `flux` + CRD (`GitRepository`, `Kustomization`, `HelmRelease`) |
| Helm (rendu, releases) | `helm template` puis apply : **pas de release Helm**, historique dans Argo CD | **vraie release Helm** (SDK) : `helm list`, hooks, tests, remédiation (retries, rollback) |
| Multi-environnement / multi-cluster | ApplicationSet (générateurs git, list, cluster, matrix) ; un Argo CD central qui pousse vers N clusters | une instance Flux **par cluster**, chacune dans son dossier `clusters/<nom>` ; dépendances (`dependsOn`) ; pas de point central |
| RBAC et multi-tenant | **AppProjects** + RBAC propre à Argo CD (SSO, rôles, sources et destinations autorisées) | **RBAC Kubernetes natif** : impersonation d'un ServiceAccount par Kustomization / HelmRelease, isolement par namespace |
| Notifications | argocd-notifications (triggers + templates : Slack, mail, webhook…) | notification-controller (`Provider` + `Alert`), et réception de webhooks (`Receiver`) |
| Automatisation des images | Argo CD Image Updater (projet séparé), write-back dans Git | intégrée : image-reflector + image-automation-controller (`ImageRepository`, `ImagePolicy`, `ImageUpdateAutomation`) |
| Courbe d'apprentissage | **douce** : l'UI rend l'état visible et pédagogique | **plus raide** : tout passe par des CRD, la CLI et les événements |

**(a) Équipe qui veut une interface visuelle partagée avec les développeurs : Argo CD.** Son UI montre l'état de chaque application (Synced / Healthy), l'arbre des ressources, le diff avec Git, les logs et les événements. Les développeurs voient eux-mêmes l'effet de leur PR sans accès `kubectl`. Le SSO et les AppProjects limitent ce que chacun peut voir ou synchroniser.

**(b) Plateforme qui gère cent clusters de façon autonome : Flux.** Chaque cluster porte sa propre instance légère, bootstrapée depuis son dossier Git. Il n'y a pas de serveur central qui détient les accès aux cent clusters : pas de point unique de défaillance ni de cible d'attaque centrale. Flux s'appuie sur le RBAC natif et se met à jour lui-même par Git, ce qui passe mieux à l'échelle. (Argo CD peut gérer de nombreux clusters avec ApplicationSet, mais l'instance centrale devient un point critique à dimensionner et à sécuriser.)

## Défis

* **Image automation** (`defi/image-automation.yaml`) : `ImageRepository` liste les tags de `traefik/whoami`, `ImagePolicy` choisit le plus récent dans `>=1.10.0 <2.0.0`, `ImageUpdateAutomation` remplace la valeur repérée par le marqueur `# {"$imagepolicy": "flux-system:whoami:tag"}` dans `apps/dev/kustomization.yaml` et **commite dans Git**. Le déploiement suit ensuite le chemin GitOps normal. Il faut bootstrapper avec `--components-extra=image-reflector-controller,image-automation-controller --read-write-key`.
* **Notifications** (`defi/notifications.yaml`) : `Provider` de type `generic` (URL du webhook dans un Secret, hors Git) et `Alert` sur `apps-dev` et la HelmRelease. Chaque réconciliation, erreur ou rollback produit un message JSON reçu sur webhook.site.
