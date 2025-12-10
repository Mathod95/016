# ArgoCD GitOps Repository

Ce dépôt contient une infrastructure GitOps complète basée sur ArgoCD avec monitoring intégré (Prometheus + Grafana) et extension de métriques.

## 🏗️ Architecture

Le projet utilise le pattern **App of Apps** d'ArgoCD pour gérer plusieurs applications de manière déclarative :

- **Root App** : Application principale qui déploie toutes les autres applications
- **Podinfo** : Application de démonstration Kubernetes
- **Kube-Prometheus-Stack** : Stack de monitoring complet (Prometheus + Grafana)
- **ArgoCD Metrics Server** : Extension pour visualiser les métriques dans l'UI ArgoCD

## 📋 Prérequis

- [KIND](https://kind.sigs.k8s.io/) - Kubernetes IN Docker
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - CLI Kubernetes
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optionnel)
- [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/) (optionnel)

## 🚀 Déploiement rapide

### 1. Créer le cluster KIND

```bash
kind create cluster --config kind-config.yaml
```

Cette configuration crée un cluster avec :
- Port mappings pour HTTP (80), HTTPS (443) et services NodePort (30080)
- Label `ingress-ready=true` pour Ingress Controller

### 2. Installer ArgoCD

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Attendre que les pods soient prêts
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 3. Configurer ArgoCD avec les extensions

```bash
# Appliquer la configuration ArgoCD (extensions + RBAC)
kubectl apply -n argocd -f bootstrap/argocd-values.yaml

# Activer le proxy extension
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.enable.proxy.extension":"true"}}'

# Redémarrer ArgoCD server pour appliquer les changements
kubectl rollout restart deployment argocd-server -n argocd
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 4. Déployer l'App of Apps

```bash
# Déployer la root application
kubectl apply -f apps/app-of-apps.yaml

# Vérifier le déploiement
kubectl get applications -n argocd
```

### 5. Accéder à ArgoCD UI

```bash
# Port-forward vers ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Accédez à https://localhost:8080 avec :
- **Username**: `admin`
- **Password**: (celui récupéré ci-dessus)

### 6. Accéder à Grafana

```bash
# Port-forward vers Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Accédez à http://localhost:3000 avec :
- **Username**: `admin`
- **Password**: `admin` (configuré dans `apps/prometheus.yaml`)

## 📊 Métriques dans ArgoCD

L'extension de métriques ajoute des onglets dans l'interface ArgoCD pour visualiser :

### Pour les Pods
- **Golden Signal** : CPU et Memory usage
- **Resource Usage** : CPU throttling, Memory working set, Memory cache
- **Network** : Bytes/Packets reçus et transmis, Erreurs réseau
- **Storage** : I/O disque (lectures/écritures), Utilisation du filesystem

### Pour les Deployments
- **Application Metrics** : HTTP latency, Error rates (4xx, 5xx), Traffic
- **Resource Usage** : CPU et Memory par deployment

## 🗂️ Structure du dépôt

```
.
├── apps/
│   ├── app-of-apps.yaml          # Root application (App of Apps)
│   ├── metrics-server.yaml       # ArgoCD metrics extension
│   ├── podinfo.yaml              # Application de démo
│   └── prometheus.yaml           # Stack Prometheus + Grafana
├── bootstrap/
│   └── argocd-values.yaml        # Configuration ArgoCD (extensions + RBAC)
├── manifests/
│   └── metrics-server/
│       ├── configmap.yaml        # Configuration des graphiques de métriques
│       ├── deployment.yaml       # Déploiement du metrics server
│       ├── service.yaml          # Service du metrics server
│       └── kustomization.yaml    # Kustomize config
├── kind-config.yaml              # Configuration du cluster KIND
└── README.md                     # Ce fichier
```

## 🔧 Configuration

### Modifier les métriques affichées

Éditez `manifests/metrics-server/configmap.yaml` pour ajouter ou modifier les graphiques de métriques.

Chaque graphique est défini par :
- `name` : Identifiant unique
- `title` : Titre affiché dans l'UI
- `description` : Description du graphique
- `graphType` : Type de graphique (`line`, `pie`)
- `queryExpression` : Requête PromQL

### Personnaliser Prometheus

Modifiez `apps/prometheus.yaml` pour :
- Ajuster les ressources (CPU, Memory)
- Changer la rétention des données
- Modifier le mot de passe Grafana
- Activer/désactiver Alertmanager

## 🛠️ Commandes utiles

### ArgoCD

```bash
# Lister toutes les applications
kubectl get applications -n argocd

# Voir les détails d'une application
kubectl describe application <app-name> -n argocd

# Forcer une synchronisation
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"sync":{}}}'

# Voir les logs d'ArgoCD
kubectl logs -n argocd deployment/argocd-server -f
```

### Prometheus

```bash
# Port-forward vers Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Vérifier les targets Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Puis ouvrir http://localhost:9090/targets
```

### Metrics Server

```bash
# Voir les logs du metrics server
kubectl logs -n argocd deployment/argocd-metrics-server -f

# Tester l'endpoint du metrics server
kubectl port-forward -n argocd svc/argocd-metrics-server 9003:9003
curl http://localhost:9003/health
```

## 🐛 Dépannage

### Les métriques ne s'affichent pas

1. Vérifier que Prometheus collecte les métriques :
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Ouvrir http://localhost:9090 et tester les requêtes PromQL
```

2. Vérifier que le metrics server est accessible :
```bash
kubectl get svc -n argocd argocd-metrics-server
kubectl logs -n argocd deployment/argocd-metrics-server
```

3. Vérifier la configuration ArgoCD :
```bash
kubectl get cm argocd-cm -n argocd -o yaml | grep -A 10 "extension.config"
kubectl get cm argocd-cmd-params-cm -n argocd -o yaml | grep proxy
```

### Applications en état "OutOfSync"

```bash
# Forcer une synchronisation
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"sync":{}}}'

# Ou via ArgoCD CLI
argocd app sync <app-name>
```

### Prometheus ne démarre pas

Vérifiez les ressources disponibles :
```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=prometheus
```

Si problème de ressources, ajustez dans `apps/prometheus.yaml` :
```yaml
resources:
  requests:
    cpu: 100m     # Réduire si nécessaire
    memory: 256Mi # Réduire si nécessaire
```

## 📚 Ressources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Metrics Extension](https://github.com/argoproj-labs/argocd-extension-metrics)
- [Kube-Prometheus-Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [KIND Documentation](https://kind.sigs.k8s.io/)

## 🔐 Sécurité

⚠️ **Attention** : Cette configuration est destinée à un environnement de développement/test.

Pour la production, pensez à :
- Changer les mots de passe par défaut (Grafana: `admin/admin`)
- Utiliser des Secrets Kubernetes au lieu de valeurs en clair
- Configurer TLS pour ArgoCD et Grafana
- Ajouter des NetworkPolicies
- Configurer des limites de ressources appropriées
- Activer l'authentification RBAC stricte

## 🤝 Contribution

Les contributions sont les bienvenues ! N'hésitez pas à :
1. Fork le projet
2. Créer une branche pour votre feature
3. Commiter vos changements
4. Pousser vers la branche
5. Ouvrir une Pull Request

## 📝 License

MIT License
