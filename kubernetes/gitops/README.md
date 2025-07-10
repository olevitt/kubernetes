# GitOps (avec argoCD)  
  
Maintenant que l'on sait déployer à peu près tout, il est temps d'automatiser les déploiements.  
De la même façon que la CI (Continuous integration / intégration continue) a revolutionné la façon de tester, builder et packager les applications, le CD (Continuous delivery / livraison en contiinue) a revolutionné la façon de déployer des applications.  

## Approche GitOps  

L'approche GitOps consiste à utiliser **Git comme source de vérité** sur ce qu'on veut déployer.  
On l'a vu, que ça soit des contrats Kubernetes (fichiers `.yaml`) ou des installations Helm (lien d'un chart + fichier `values.yaml`), tout ce qu'on déploie sur Kubernetes est sous la forme de code (`as code`). Il est donc tout à fait adapté de les versionner sur Git.  
Tout comme pour la CI, on va mettre en place des robots qui vont surveiller en permanence le dépôt git et ré-appliquer, à chaque changement, les nouveaux contrats (donc faire `kubectl apply -f` et/ou `helm upgrade`) à chaque commit. 
L'avantage principal est, comme pour la CI, que l'on automatise les tâches fastidieuses de déploiement, on réduit les risques d'erreur humaine et on assure une traçabilité complète de tous les changements gràce au versionning git.  

## argoCD 

[argoCD](https://github.com/argoproj/argo-cd) est un moteur GitOps. Son rôle est de scanner les dépôts Git dont vous lui donnez connaissance et de ré-appliquer les manifests dans le cluster à chaque changement.  
argoCD supporte à la fois les contrats Kubernetes bruts (fichiers `.yaml`) mais aussi les installations Helm (chart + fichier `values.yaml`).  

## Installation d'argoCD  

argoCD propose un [chart helm](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd) facilitant l'installation.  
argoCD est découpé en différents pods :  
    * argocd-application-controller : moteur de logique argocd (celui qui mène la danse)
    * argocd-dex-server : moteur d'authentification
    * argocd-redis : cache distribué pour le partage des informations entre les pods
    * argocd-repo-server : chargé de synchroniser (`git pull`) les repositories
    * argocd-server : interface graphique de gestion (à ingressifier)

## Pattern operator et CRD  

argoCD fonctionne selon le pattern [operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).  
Le pattern operator consiste à faire tourner un pod en permanence (appelé `operator`) qui scanne l'apparition ou la modification d'une certaine ressource Kubernetes (un certain `Kind`) et réagit en conséquence, en général pour déployer (/ modifier) une application définie dans cette ressource.  
En général, un opérator travaille sur un `Kind` qu'elle a lui même inventé (et non pas un Kind installé par défaut dans Kubernetes). On parle alors de `CRD` (Custom Resource Definition).  

Concrètement, `argocd` introduit les concepts / objets (CRD) suivants :  
```
Application (applications.argoproj.io)                                   
ApplicationSet (applicationsets.argoproj.io)                                
AppProject (appprojects.argoproj.io)                                    
ArgocdExtension (argocdextensions.argoproj.io)
```  

Ces CRD sont en général installées en même temps que argoCD lui même (https://github.com/argoproj/argo-helm/blob/dd206e8e3074775472129e299cf0a26a7873930d/charts/argo-cd/values.yaml#L33).  

## CRD Application  

Comme vu au paragraphe précédent, argoCD définit un objet (CRD) `Application`.  
Pour créer (ou modifier / supprimer) une application dans argoCD il faut créer (ou modifier / supprimer) un objet `Application`.  
C'est faisable soit via l'interface graphique argocd, soit en appliquant (`kubectl apply -f application.yaml`) directement le yaml d'un kind `Application`. Note : argoCD fournit également une [CLI](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd/).  
Voici un exemple de CRD `Application` :  
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
spec:
  project: default
  source:
    repoURL: https://github.com/username/repo.git
    targetRevision: HEAD
    path: apps/mon-app
  destination:
    server: https://kubernetes.default.svc
    namespace: mon-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```  

Une fois appliqué, cet objet `Application` va dire à argoCD d'appliquer / ré-appliquer le contenu (fichiers `.yaml` et installations helm si présence d'un `Chart.yaml`) du dossier `apps/mon-app` du repo `https://github.com/username/repo.git` dans le namespace `mon-app` de façon automatique et en crééant, si besoin, le namespace.  

