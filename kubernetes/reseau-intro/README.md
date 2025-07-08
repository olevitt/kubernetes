# Réseau : intro

Maintenant que l'on a réussi à déployer un pod, on aimerait bien pouvoir y accéder niveau réseau.  
Afin de pouvoir y accéder, il faut comprendre ce qu'il se passe niveau réseau.  

## Rappel : architecture

![](../kesako/img/architecture.png)  

Votre pod tourne donc quelque part, sur un des noeuds workers.  

## Pour le débug : port-forward  

Dans la suite, on va décortiquer le réseau et, dans les chapitres suivants, voir comment on peut exposer notre pod à l'intérieur et à l'extérieur du cluster.  
Mais, quand on veut simplement tester la connectivité du pod, il existe une commande très utile qui créé un **tunnel temporaire direct** entre votre machine (celle qui exécute la commande `kubectl`) et le pod en passant par l'APIServer.  
Cette commande ne doit servir que pour du débug ou du testing, pas en production ! On verra dans les chapitres suivants des façons propres et stables d'exposer nos applications.  

On récupère le nom du pod :  
```
gon@laboitemagique:~$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
nginx-627544-5cc45d7bd6-nsxvz   1/1     Running   0          6d21h
```

On créé le tunnel temporaire : 
```
gon@laboitemagique:~$ kubectl port-forward nginx-627544-5cc45d7bd6-nsxvz 9000:8080  
Forwarding from 127.0.0.1:9000 -> 8080
Forwarding from [::1]:9000 -> 8080
```  
tant que cette commande est active, tous les flux vers `localhost:9000` seront transmis (via l'APIServer) directement au pod sur le port `8080`.  

> [!WARNING]  
> **Cette commande est bien pratique pour le débug / testing mais n'est pas un moyen stable d'exposer une application.**

## Réseau des pods  

On l'a vu, la conteneurisation vise à isoler entièrement les processus. Il y a donc une isolation de tous les aspects du processus (système de fichiers, mémoire ...) et donc une isolation du réseau.  
Pour cela, les moteurs de conteneurisation crééent en général un réseau virtuel dans lequel vont évoluer les différents conteneurs.  
En Kubernetes, **chaque pod possède sa propre IP** dans le réseau des pods.  
Ce mécanisme (CNI : Container Network Interface) est délégué à un provider CNI dont il existe de nombreuses implémentations. Le plus connu est probablement [calico](https://github.com/projectcalico/calico) mais on peut aussi citer Canal, Weave, Flannel, Cilium ...  
Le rôle du CNI provider est de créer et maintenir le réseau privé des pods et d'allouer dynamiquement une IP à chaque pod qui se créé.  
Le provider CNI est en général installé dans le namespace `kube-system`. Vous pouvez jeter un oeil dans ce namespace système via la commande `kubectl get pods -n kube-system`

## IP d'un pod

Pour obtenir l'IP d'un pod, on peut par exemple utiliser la commande `kubectl get pods -o wide`.  
Exemple :  
```
gon@laboitemagique:~$ kubectl get pods -o wide
NAME                             READY   STATUS        RESTARTS   AGE    IP              NODE     NOMINATED NODE   READINESS GATES
nginx-627544-5cc45d7bd6-nsxvz   1/1     Running       0          6d8h   10.233.112.84   boss4    <none>           <none>
```  

Ici, mon pod a l'IP `10.233.112.84`.  

> [!WARNING]  
> **Cette IP n'est pas statique ! Elle n'est "allouée" au pod que pendant la durée de sa vie.**   
Une fois qu'un pod meurt, son IP retourne dans le pool des IP disponibles et peut même être réallouée à un autre pod complètement différent ultérieurement.  
Vous pouvez en faire l'expérience en supprimant manuellement le pod `kubectl delete pod nginx-627544-5cc45d7bd6-nsxvz`  


## Utilisation de l'IP d'un pod  

L'IP d'un pod est une IP privée (IP en `10.` = IP privée) du réseau des pods. Elle n'a donc de sens que si on se positionne soi-même sur ce réseau c'est à dire en étant dans un pod.  
Par défaut, Kubernetes n'applique aucune restriction sur le traffic inter-pods et inter-namespaces (il existe des moyens de rajouter des restrictions via par exemple les `NetworkPolicy`).  
On peut donc, depuis un pod du cluster, requêter un autre pod du cluster (y compris d'un autre namespace) en utilisant son IP actuelle.  

```
gon@laboitemagique:~$ kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE
olivier-6fd84b8877-stw85         1/1     Running   0          12s     10.233.122.235   boss7
olivier-6fd84b8877-tgdxw         1/1     Running   0          12s     10.233.112.205   boss4
```  

```
gon@laboitemagique:~$ kubectl exec -it olivier-6fd84b8877-stw85 -- sh
~ $ curl 10.233.112.205:9898
{
  "hostname": "olivier-6fd84b8877-tgdxw",
  "version": "6.9.0",
  "revision": "fb3b01be30a3f353b221365cd3b4f9484a0885ea",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.9.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.24.3",
  "num_goroutine": "6",
  "num_cpu": "72"
} 
```  

On a bien réussi, depuis le premier pod, à contacter le deuxième pod :)  

