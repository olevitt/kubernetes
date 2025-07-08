# Réseau : exposition extra-cluster

Maintenant qu'on sait exposer notre application à l'intérieur du cluster, il est temps de s'intéresser à l'exposition vers l'extérieur du cluster.  
Il existe 4 grandes façons d'exposer une application Kubernetes à l'extérieur :  
  * `HostNetwork` / `HostPort` : permet de rompre l'isolation réseau en faisant utiliser au pod directement le réseau de l'hôte
  * `Service` de type `NodePort` : permet d'exposer le `Service` sur un port de tous les noeuds du cluster
  * `Service` de type `LoadBalancer` : permet d'attribuer une IP publique au `Service` (en plus de son IP interne)  
  * Reverse proxy-fier le service (uniquement pour le HTTP(S)) : permet d'enregistrer automatiquement le service auprès d'un reverse-proxy (`Ingress controller`) avec un nom de domaine  

**Dans l'immense majorité des cas, on privilégiera la solution de reverse-proxification et on privilégiera l'exposition de service HTTP(S). Les services non-HTTP (e.g postgres) ayant en général vocation à rester en intra-cluster.**  

## `HostNetwork` / `HostPort`  

`HostNetwork` casse l'isolation réseau en faisant utiliser au pod directement le réseau de l'hôte et non pas le réseau des pods (CNI). Ainsi, un conteneur d'un pod en `HostNetwork` qui bind un port bindera directement le port du système hôte, comme si le processus avait été executé directement sur le système hôte, sans isolation.  
Du coup, si le port est déjà bindé à un autre pod `HostNetwork` ou à un autre processus du système hôte, le bind échouera. C'est similaire à `--network=host` en Docker.  
`HostPort` est dans le même esprit mais en plus léger : le pod est quand même dans le réseau des pods mais Kubernetes configure `iptables` sur le noeud pour rediriger le traffic reçu vers le conteneur.

> [!WARNING]  
> `HostNetwork` / `HostPort` étant construit sur une rupture de l'isolation, leur utilisation est fortement découragée sauf pour des usages très très précis et est en général réservé aux administrateurs de l'infrastructure (mises en place de restrictions de sécurité via des `Admission controllers`). 