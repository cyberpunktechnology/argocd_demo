# argocd_demo

Démo ArgoCD

## Installation du Setup en local

### Environnement

* MacOS X
* Colima + K3s

### Prérequis

```bash
brew install mise colima docker 
```

```bash
mise install argo argocd argo-rollouts helm
mise use --global argo@latest
mise use --global argocd@latest
mise use --global argo-rollouts@latest
mise use --global helm@latest
```

### Installation du cluster avec ArgoCD

#### Nous démarrons une instance colima sous le nom de profile `argocd` afin d'éviter d'utiliser le profil par défaut.

```bash
colima start --profile argocd --kubernetes --k3s-arg='"--disable=metrics-server,traefik"' \
--cpu 4 --memory 8 --disk 100  
```

#### Déclarer les hosts internes pour tester dans votre navigateur

```bash
echo "127.0.0.1 demo.local argocd.local podinfo.local canary.local"|tee -a /etc/hosts
```

#### Install de l'ingress Nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.hostNetwork=true \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443 \
  --set controller.kind=DaemonSet
```

#### Vérification que le `ingress-nginx` est bien installé

```bash
kubectl apply -f ingress-nginx-test/test.yaml
```

Ce curl doit afficher la page de bienvenue de nginx avec en `title` : `Welcome to nginx!`

Vous pouvez bien sûr vérifier via <https://demo.local:30443/>

```bash
curl https://127.0.0.1:30443 --insecure -H 'Host: demo.local'
```

puis clean l'environnement

```bash
kubectl delete -f ingress-nginx-test/test.yaml
```

#### Installation d'ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
    --namespace argocd --create-namespace \
    --set server.extraArgs={--insecure} \
    --set server.ingress.enabled=true \
    --set server.ingress.ingressClassName="nginx" \
    --set server.ingress.hostname="argocd.local" \
    --set-string configs.cm."exec\.enabled"="true"
```

Note: l'option `configs.cm.exec.enabled=true` permet d'ouvrir un terminal dans un pod depuis l'interface ArgoCD.

Maintenant l'UI Web d'ArgoCD est dispo via <http://argocd.local:30080>

### ArgoCD

Pour récupérer le mot de passe du user `admin` :

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode
echo
```

Authentification ArgoCD sur le terminal :

```bash
ARGO_ADMIN_PASS=$(kubectl get secret argocd-initial-admin-secret  -n argocd -o jsonpath="{.data.password}" | base64 --decode)
argocd login --insecure argocd.local:30080 --grpc-web --username admin --password "${ARGO_ADMIN_PASS}"
```

Note : l'authentification expire régulièrement

Ajouter le repo Github : 

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_argocd_demo # wihtout passphrase
```

Ajouter la clé publique via le fichier `~/.ssh/github_argocd_demo.pub` au niveau de `Settings/Deploy Keys` de votre repo.

```bash
argocd repo add git@github.com:cyberpunktechnology/argocd_demo.git --ssh-private-key-path ~/.ssh/github_argocd_demo

# vérifier via
argocd repo list
```
## Demonstration

### Simple app

### Hook Presync

### Git generators

Nous avons besoin de créer un secret pour s'authentifier sur l'API de Github, il faut un PAT avec les droits : 

* repo:status
* public:repo
* read:project

```bash
kubectl create secret generic github-creds -n argocd --from-literal=token=$(echo ${GITHUB_ARGOCD_TOKEN})
kubectl label secret github-creds -n argocd argocd.argoproj.io/secret-type=scm-creds
```

## Documentations externes

Nginx ingress controller : 

* <https://docs.nginx.com/nginx-ingress-controller/install/helm/open-source/>
* <https://docs.nginx.com/nginx-ingress-controller/install/helm/parameters/>

ArgoCD :

* <https://argo-cd.readthedocs.io/en/latest/>