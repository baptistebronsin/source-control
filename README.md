# Source Control

## Auteurs
- Baptiste Bronsin
- Pauline Contat
- Giada Flora De Martino

## Attendus
- Installer ArgoCD via Helm
- Installer AirFlow avec kubernetes executor via ArgoCD
- Proposer un pipeline de déploiement pour AirFlow + Dags (postgres en H.A.)

## Prérequis
- Posséder une machine avec un accès à internet (une machine virtuelle par exemple)

## Installation
### Kubernetes
| Mot clé   | Signification                    |
|-----------|----------------------------------|
| `ter-vm`  | Terminal de la machine virtuelle |
| `ter-loc` | Terminal de la machine locale    |

(Ces mots clés ne seront utilisés que pour cette partie.)

1. Installation du moteur de Kubernetes _k3s_ sur la machine virtuelle dans `ter-vm`
```bash
curl -sfL https://get.k3s.io | sh -
```

2. Récupérer les informations de connexion à notre nouveau cluster kubernetes dans `ter-vm`
```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

3. Créer un fichier `~/.kube/config_vm` dans `ter-loc` et y copier le contenu du fichier
```bash
nano ~/.kube/config_vm
```

4. Modifier le contenu de notre fichier
```txt
server: https://127.0.0.1:6443 -> server: https://IP_VM:6443
```

5. Modifier la variable d’environnement `KUBECONFIG` dans `ter-loc`
```bash
export KUBECONFIG=~/.kube/config_vm
```

6. Vérifier la connexion à notre cluster kubernetes dans `ter-loc`
```bash
kubectl get node
```

### Helm
Installer Helm sur la machine locale ([documentation officielle](https://helm.sh/docs/intro/install))
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```
```bash
chmod 700 get_helm.sh
```
```bash
./get_helm.sh
```

## Déployer et configurer ArgoCD
[Documentation suivie](https://medium.com/@aavulasaikiran/argocd-installation-using-helm-charts-7711863ced65)

1. Cloner le dépôt ArgoCD
```bash
git clone https://github.com/argoproj/argo-helm.git
```

2. Naviger dans le répertoire du dépôt cloné
```bash
cd argo-helm/charts/argo-cd/
```

3. Créer un namespace kubernetes `myargo`
```bash
kubectl create ns myargo
```

4. Mettre à jour les dépendances Helm
```bash
helm dependency up
```

5. Installer ArgoCD
```bash
helm install myargo . -f values.yaml -n myargo
```

6. Accéder au service ArgoCD depuis la machine locale
```bash
kubectl port-forward service/myargo-argocd-server 9000:80 -n myargo
```

7. Récupérer le mot de passe par défaut
```bash
kubectl -n myargo get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```
| Label  | valeur           |
|--------|------------------|
|username|admin             |
|password|`valeur retournée`|

8. Se connecter à ArgoCD via le navigateur à l'adresse `http://localhost:9000` avec les identifiants récupérés

## Déployer AirFlow avec Kubernetes Executor
1. Se placer à la racine du projet et exécuter la commande suivante
```bash
kubectl apply -f application.yaml
```

2. Accéder au service AirFlow depuis la machine locale
```bash
kubectl -n myargo port-forward svc/airflow-webserver 8080:8080
```

3. Se connecter à AirFlow via le navigateur à l'adresse `http://localhost:8080` avec les identifiants suivants

| Label  | valeur         |
|--------|----------------|
|username|admin           |
|password|admin           |

## Synchroniser les DAGs

1. Générer une clé SSH
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

2. Créer le secret dans le cluster Kubernetes
```bash
kubectl create secret generic airflow-ssh-secret   --from-file=gitSshKey=cle_privee   --from-file=known_hosts=known_hosts   --from-file=id_ed25519.pub=cle_publique.pub   -n myargo
```
**Attention**: Pensez à remplacer `cle_privee`, `known_hosts` et `cle_publique.pub` par les chemins correspondants.

3. Ajouter la clé public générée au dépôt Github dans les `Deploy keys`

4. Rafraîchir manuellement l'application Airflow depuis l'interface ArgoCD

5. Attendre 5min et les DAGs devraient apparaître dans l'interface web Airflow