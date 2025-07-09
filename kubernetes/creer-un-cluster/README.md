# Créer un cluster multi-noeuds
  
On se propose de créer un cluster multi-noeuds (multi-machines).  

## Choix de la distribution Kubernetes  

Bien qu'il soit possible d'installer manuellement Kubernetes en installant et configurant chaque composant un par un (Kubelet, APIServer, ETCD ...) ça se rélève très vite fastidieux et complexe à maintenir et faire évoluer.  
Si vous souhaitez quand même tenter l'aventure, Kelsey Hightower, un des développeurs de Kubernetes, propose un guide étape par étape en partant de 0 jusqu'à un cluster Kubernetes fonctionnel : [Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way).  

Dans notre cas nous allons utiliser une distribution Kubernetes c'est à dire un repackaging de Kubernetes offrant une facilité d'installation, de configuration ainsi qu'un certain nombre de bonus (plugins, applications préinstallées ...).  
Certaines distributions sont plus complètes que d'autres et font plus ou moins de choix. Tout comme pour les frameworks en développement, on parle de distributions plus ou moins `opinionated` (dogmatiques en français) en fonction d'à quoi point elles font des choix à votre place ou non.  

Dans la suite on va utiliser la distribution [k3s](https://k3s.io/), légère, rapide et facile à installer.  

> [!WARNING]  
> k3s ne fonctionne que sous Linux. Certaines distributions sont compatibles avec d'autres OS mais pas k3s.  

> [!NOTE]  
> Chaque distribution a sa propre manière de s'installer. Par exemple : [Kubespray](https://github.com/kubernetes-sigs/kubespray) repose sur `ansible`, k3s fournit un script `one-liner`, docker desktop cache la vérité et offre une simple case à cocher ...


## Installation du controlplane  

[Quickstart k3s](https://docs.k3s.io/quick-start)

A minima, un cluster doit avoir au moins un controlplane (rappel : le controlplane contient l'APIServer).  
k3s fournit un script `one-liner` pour installer les différents composants prérequis et configurer le controlplane (à exécuter sur le futur noeud controlplane) :  

```
curl -sfL https://get.k3s.io | sh - 
# Check for Ready node, takes ~30 seconds 
sudo k3s kubectl get node 
```  

Félicitations ! Votre cluster est prêt avec un unique controlplane et le `kubeconfig` est disponible à cet endroit `/etc/rancher/k3s/k3s.yaml`.  

```
After running this installation:

The K3s service will be configured to automatically restart after node reboots or if the process crashes or is killed
Additional utilities will be installed, including kubectl, crictl, ctr, k3s-killall.sh, and k3s-uninstall.sh
A kubeconfig file will be written to /etc/rancher/k3s/k3s.yaml and the kubectl installed by K3s will automatically use it
```

## Raccorder des workers au cluster  

Maintenant que le cluster est opérationnel, on va pouvoir rajouter des noeuds workers :  
Sur chaque worker à raccorder : `curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -`  
Avec `K3S_URL` qui correspond à l'URL de l'APIServer (donc l'IP du controlplane)  
`K3S_TOKEN` qui est le token d'enregistrement (pour éviter que n'importe qui puisse enregistrer un worker au cluster). La valeur de `K3S_TOKEN` est disponible sur le controlplane dans le fichier `/var/lib/rancher/k3s/server/node-token`  

Dès qu'un worker rejoint le cluster, il est visible via `kubectl get nodes` et est prêt à recevoir des charges de travail (pods)  

## Constater la répartition du travail  

Créez et appliquez un ou plusieurs `Deployments`. `kubectl get pods -o wide` vous permet de voir, pour chaque pod créé, sur quel noeud il a été affécté (schedulé). Sur le noeud correspondant, vous pouvez exécuter la commande `crictl ps` afin de lister les conteneurs qui tournent. Vous devriez voir les conteneurs correspondant à chaque pod schedulé sur le noeud.  

> [!NOTE]  
> Kubernetes est compatible avec n'importe quel container engine à partir du moment où ce container engine implémente l'interface `CRI` (Container Runtime Interface). L'outil `crictl` est compatible avec n'importe quel container engine qui implémente l'interface `CRI` et propose une syntaxe assez proche de la syntaxe de `docker`, vous ne serez donc pas trop dépaysés.   
Docker ne respecte pas l'interface `CRI`, c'est pour cela que le support natif de Docker en tant que container engine a été déprécié dans la version `1.20` fin 2020 (cf [cet article](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/)). 


## Supprimer un worker  

Si vous voulez retirer un noeud worker vous pouvez utiliser la commande `kubectl delete node xxxxx`.  
Attention : si vous retirez un noeud mais que k3s tourne toujours sur le worker, il va tenter de se re-enregistrer de nouveau au cluster

