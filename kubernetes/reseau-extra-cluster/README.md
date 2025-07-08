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

## `Service` de type `NodePort`  

On a vu au chapitre précédent (intra-cluster) que les services étaient par défaut de type `ClusterIP` et obtenaient une IP interne au cluster.  
Il existe aussi des `Service` de type `NodePort` qui, en plus de faire ce que font les `Service` de type `ClusterIP`, sont aussi exposés sur TOUS les noeuds du cluster sur un port spécifique (entre 30000 et 32767).  
Le port peut être choisi explicitement ou laissé vide pour être attribué aléatoirement (dans la plage 30000 - 32767) par Kubernetes. Attention : si vous choisissez explicitement le numéro du port, il faut être sûr que le port n'est pas déjà utilisé par un autre `NodePort` (ou par un applicatif sur un des noeuds).  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: olivier
  ports:
  - port: 80
    targetPort: 9898
    nodePort: 31000
```

> [!WARNING]  
> Ce mode n'est pas vraiment recommandé en pratique et ne doit être utilisé qu'avec modération car il multiplie les points d'entrée dans le cluster ce qui en complexifie la gestion. De plus, le port exposé sera dans une plage non-standard (30000 - 32767)   

## `Service` de type `LoadBalancer`  

On a vu au chapitre précédent (intra-cluster) que les services étaient par défaut de type `ClusterIP` et obtenaient une IP interne au cluster.  
Il existe aussi des `Service` de type `LoadBalancer` qui, en plus de faire ce que font les `Service` de type `ClusterIP`, cherchent à obtenir une IP publique (en plus de la `ClusterIP`).  
Les `Service` de type `LoadBalancer` ne fonctionnent que sur des clusters ayant un provider de LoadBalancer installé (par exemple [MetalLB](https://metallb.io/)). Si il n'y a pas de provider installé (ou si le pool d'IP publique est épuisé / l'IP demandé n'est pas disponible), le service ne se verra jamais attribuer d'IP publique et son `external-ip` restera en `<pending>`.  

> [!NOTE]  
> Les clusters managés des cloud providers (GKE, AKS, EKS ...) viennent tous avec un provider préinstallé. La création d'un `Service` `LoadBalancer` sur un cluster managé aboutit donc (l'opération peut prendre quelques minutes) à la création d'une IP publique et à son affectation au `Service` `LoadBalancer`. Attention : la plupart des cloud providers facturent la réservation d'une IP publique et les flux réseau entrants / sortants.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: olivier
  ports:
  - port: 80
    targetPort: 9898
    loadBalancerIP: x.x.x.x
```

> [!WARNING]  
> Ce mode n'a pas vocation à être généralisé à de nombreuses applications sur un cluster car chaque LoadBalancer a un coût (mise en place, maintenance, réservation d'une IP publique). En général, on se limite à un seul service LoadBalancer pour exposer le reverse proxy (cf paragraphe suivant).  

## Reverse proxification (HTTP(S) uniquement)  

Le concept de [reverse proxy](https://www.cloudflare.com/fr-fr/learning/cdn/glossary/reverse-proxy/) n'est pas spécifique à Kubernetes mais Kubernetes amène des API pour gérer un reverse proxy à l'intérieur du cluster (`Ingress controller`) avec une gestion dynamique des enregistrements (`Ingress`).  
Le kind `Ingress` correspond à un enregistrement auprès du reverse proxy.  
Le reverse proxy en lui même s'appelle un `Ingress controller`.  
Il existe de multiples implémentations d'Ingress controllers. La plus populaire est probablement [`ingress-nginx`](https://github.com/kubernetes/ingress-nginx), basée sur le logiciel [nginx](https://nginx.org/).  

L'installation d'un `Ingress controller` varie en fonction du type de cluster (on-premise, cloud, managé ...) mais globalement in-finé il s'agit d'un (ou plusieurs) pods qui centralisent les flux HTTP(S) entrants dans le cluster et les dispatchent sur vos `Service` en fonction du nom d'hôte choisi. La façon dont l'ingress controller est lui même exposé à l'extérieur dépend de l'installation et du contexte (on peut le faire via un HostPort / HostNetwork, via un service NodePort ou via un LoadBalancer).  

Une fois l'Ingress controller installé et configuré (par exemple avec un certificat TLS), on peut s'enregistrer auprès de lui en créant des objets de kind `Ingress`.  

> [!NOTE]  
> Un cluster ne possède en général qu'un seul Ingress controller (éventuellement 2 ou 3 si on veut segmenter les flux d'entrée) mais possède des dizaines voire des centaines d'Ingress.  

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myingress
spec:
  rules:
  - host: monapp.mondomaine.fr
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: myapp
            port: 
              number: 80
```

`kubectl apply -f ingress.yaml`  

Une fois l'ingress créé, l'ingress controller va le détecter et ajouter le mapping host (`monapp.mondomaine.fr`) vers `Service` (`myapp:80`).  
Au final, on a la stack ingress -> service -> pod.  

Félicitations ! Vous pouvez maintenant accéder à votre application sur `http://monapp.mondomaine.fr` (en supposant que les DNS pointent bien vers votre cluster et que le ingress controller est bien configuré pour recevoir le traffic).  

> [!NOTE]  
> Vous pouvez mettre ce que vous voulez en nom d'hôte dans l'ingress (par exemple `google.com`) mais, si les DNS ne sont pas configurés pour pointer vers votre cluster, vous ne risquez pas vraiment de recevoir du traffic sur votre Ingress.  