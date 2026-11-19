# TP20 : GitOps avec Flux CD (et comparaison avec Argo CD)

Flux déploie depuis ce dépôt la même application qu'au TP19 (Kustomize) et le chart Helm du TP15 (HelmRelease). Flux n'a pas d'interface : ce sont des contrôleurs Kubernetes pilotés par des ressources (GitRepository, Kustomization, HelmRelease) et par la CLI `flux`.
Réponses et tableau comparatif : [REPONSES.md](REPONSES.md).

```
.
├── clusters/lab/
│   ├── flux-system/                ← écrit par flux bootstrap (ne pas modifier)
│   ├── apps.yaml                   ← Kustomization Flux apps-dev → ./apps/dev
│   └── helm-whoami.yaml(.example)  ← HelmRelease du chart TP15 (Partie 4)
├── apps/
│   ├── base/                       ← Deployment + Service whoami
│   └── dev/                        ← namespace whoami-flux, 2 replicas, v1.10.1
├── charts/whoami/                  ← chart du TP15 (Partie 4)
└── defi/                           ← image automation, notifications
```

Namespaces **distincts** de ceux d'Argo CD (TP19) : `whoami-flux` et `whoami-helm`. Deux outils GitOps ne doivent jamais gérer les mêmes ressources.

## 0. Prérequis

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
flux check --pre
```

Jeton GitHub : *Settings → Developer settings → Personal access tokens → Tokens (classic)* → scope **repo**.

## 1. Partie 1 : bootstrap

```bash
export GITHUB_TOKEN=<token>
flux bootstrap github --owner=soulaymahableni --repository=tp20-flux \
  --branch=main --path=clusters/lab --personal
kubectl -n flux-system get pods
clear
flux get sources git
flux get kustomizations
```

**Capture `flux-bootstrap.png`** : fin du bootstrap (`all components are healthy`) puis `flux get kustomizations` (`flux-system … True … Applied revision: main@sha1:…`).

```bash
cd ~ && git clone https://github.com/soulaymahableni/tp20-flux.git && cd tp20-flux
ls clusters/lab/flux-system/          # gotk-components.yaml gotk-sync.yaml kustomization.yaml
```

## 2. Partie 2 : application Kustomize

```bash
cp -rn ~/flux-starter/. ~/tp20-flux/          # -n : n'écrase pas clusters/lab/flux-system
cd ~/tp20-flux
git add . && git commit -m "feat: application whoami (apps-dev)" && git push
flux reconcile source git flux-system
flux get kustomizations --watch               # apps-dev True, puis Ctrl+C
kubectl -n whoami-flux get all
```

Dérive :

```bash
clear
kubectl -n whoami-flux scale deploy whoami --replicas=4
kubectl -n whoami-flux get deploy whoami -w   # 4, puis 2 au plus tard 1 minute après, Ctrl+C
```

**Capture `flux-drift.png`**.

Erreur :

```bash
sed -i 's/^resources:/ressources:/' apps/dev/kustomization.yaml
git commit -am "test: faute dans apps/dev" && git push
flux reconcile source git flux-system
sleep 20
clear
flux get kustomizations
flux logs --kind=Kustomization --name=apps-dev --tail=10
kubectl -n flux-system describe kustomization apps-dev | tail -15
```

**Capture `flux-error.png`** : `apps-dev False … kustomize build failed: … invalid Kustomization: json: unknown field "ressources"`.

```bash
git revert --no-edit HEAD && git push
flux reconcile kustomization apps-dev --with-source
```

## 3. Partie 3 : suspendre, reprendre, supprimer

```bash
clear
flux suspend kustomization apps-dev
flux get kustomizations                                       # SUSPENDED True
kubectl -n whoami-flux scale deploy whoami --replicas=3
sleep 70; kubectl -n whoami-flux get deploy whoami            # toujours 3
flux resume kustomization apps-dev
kubectl -n whoami-flux get deploy whoami                      # revient à 2
```

**Capture `flux-suspend.png`**.

Prune :

```bash
git rm apps/base/service.yaml && sed -i '/service.yaml/d' apps/base/kustomization.yaml
git commit -m "test: suppression du Service" && git push
flux reconcile kustomization apps-dev --with-source
kubectl -n whoami-flux get svc                                # No resources found
git revert --no-edit HEAD && git push
flux reconcile kustomization apps-dev --with-source
kubectl -n whoami-flux get svc                                # whoami revenu
```

## 4. Partie 4 : HelmRelease

Copier le chart (voir `charts/README.md`), puis :

```bash
mv clusters/lab/helm-whoami.yaml.example clusters/lab/helm-whoami.yaml
git add . && git commit -m "feat: HelmRelease whoami (chart TP15)" && git push
flux reconcile source git flux-system
sleep 60
flux get helmreleases -A
helm list -n whoami-helm
helm history whoami -n whoami-helm
```

Valeur cassée :

```bash
sed -i 's/tag: v1.10.1/tag: v9.9.9/' clusters/lab/helm-whoami.yaml
git commit -am "test: whoami v9.9.9 via Helm" && git push
flux reconcile source git flux-system
# attendre 4 à 5 minutes (1 essai + 2 retries de 1 min chacun, puis rollback)
clear
flux get helmreleases -A
helm history whoami -n whoami-helm
```

**Capture `helmrelease.png`** : révisions `failed` puis `Rollback to …`, et la HelmRelease en échec de mise à jour mais avec l'ancienne version servie.

```bash
git revert --no-edit HEAD && git push
flux reconcile helmrelease whoami --with-source
```

## 5. Défis

Voir `defi/` et REPONSES.md.

## 6. Nettoyage

```bash
flux uninstall --silent           # retire Flux (les ressources déployées restent ; supprimer les namespaces si besoin)
kubectl delete ns whoami-flux whoami-helm --ignore-not-found
```

## Captures

(à compléter)
