# Réseau : exposition intra-cluster

On a réussi à déployer un pod et à obtenir et utiliser son IP temporaire.  
Il est temps maintenant d'exposer proprement notre application à l'intérieur du cluster.  
Les enjeux principaux d'une exposition propre en intra-cluster sont les suivants :  
  * **Stabilité** : on a vu que l'IP d'un pod n'était pas statique. On ne peut donc pas s'appuyer dessus pour configurer les autres applications (si vous dites à votre app Java que son postgres est sur `10.233.122.235` puis que le postgres redémarre et que son IP n'est plus allouée ou pire que son IP est allouée à un tout autre pod, ça va pas le faire)  
  * **Load balancing** : On a vu qu'un deployment pouvait avoir plusieurs replicas et donc correspondre à plusieurs pods simultanément. Chacun de ces pods (en supposant qu'ils soient en bonne santé) est apte à recevoir du traffic. On souhaite donc ne pas en cibler un en particulier mais répartir relativement équitablement et en temps réel la charge entre les replicas vivants.  
  * **Practicité** : Les IP c'est rigolo mais c'est plus pratique de dire à une application Java que le postgres est sur le nom d'hôte `monpostgres` que sur `10.233.122.235`. Clin d'oeil, clin d'oeil DNS.  

On va voir un système permettant d'adresser ces 3 enjeux à la fois : les `Service`.  

## Service

Un `Service` (kind Kubernetes) est un point d'entrée réseau qui va répartir les connexions qu'il reçoit sur les différents pods en bonne santé (`ready`).  
Supposons que l'on ait le deployment créé dans les chapitres précédents avec 2 replicas :  

```sh
gon@laboitemagique:~$ kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE
olivier-6fd84b8877-stw85         1/1     Running   0          12s     10.233.122.235   boss7
olivier-6fd84b8877-tgdxw         1/1     Running   0          12s     10.233.112.205   boss4
```  

On peut créer le `Service` suivant :  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 9898
```  

Décortiquons ce yaml :  
  * `kind: Service` : Hop, on découvre un nouvel objet Kubernetes  
  * ```yaml
      selector:
        app: myapp
    ```
    Ce [selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) indique que les pods vers lesquels ce service loadbalance sont tous les pods qui ont le label `app: myapp`. Pour lister les objets correspondant à un selector (donc les objets ayant un certain label) : `kubectl get pods -l app=olivier`  
  * `port: 80` : port d'entrée du `Service`. Vous pouvez choisir ce que vous voulez, indépendamment du port d'écoute du conteneur.  
  * `targetPort: 9898` : port vers lequel le `Service` va loadbalancer. **Vous devez mettre le port d'écoute du conteneur**

On créé ce `Service` : `kubectl apply -f service.yaml`  

```sh
gon@laboitemagique:~$ kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
myapp           ClusterIP   10.233.1.185   <none>        80/TCP    1s
```

Notre service est opérationnel et a obtenu un IP interne au cluster (`10.233.1.185`).  
On peut maintenant tester ce service :  

```sh
gon@laboitemagique:~$ kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE
olivier-6fd84b8877-stw85         1/1     Running   0          51m     10.233.122.235   boss7
olivier-6fd84b8877-tgdxw         1/1     Running   0          51m     10.233.112.205   boss4
```  
```sh
gon@laboitemagique:~$ kubectl exec -it olivier-6fd84b8877-stw85 -- sh
~ $ curl http://10.233.1.185:80
{
  "hostname": "olivier-6fd84b8877-stw85",
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
~ $ curl http://10.233.1.185:80
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

La première requête a été servie par un des pods (`"hostname": "olivier-6fd84b8877-stw85"`) tandis que la seconde a été servie par l'autre (`"hostname": "olivier-6fd84b8877-tgdxw"`).  
Le loadbalancing fonctionne donc bien ! Et l'adresse IP du `Service` (`10.233.1.185`) est stable peu importe la mort des pods sous-jacents (testez par vous même de changer le nombre de replicas / `delete` les pods).  

> [!NOTE]  
> Le `Service` n'offre pas de garantie sur la précision du loadbalancing. En gros il fait du `round-robin` (1 puis 2 puis 3 puis 1 puis 2 puis 3) mais il peut exister des optimisations (exemple le service privilégie les pods qui sont sur le même noeud que l'origine de la connexion) ou simplement le fait que d'autres connexions aient lieu en parallèle. De toute façon ce qui compte c'est que l'équilibre ait lieu à l'échelle c'est à dire sur un grand nombre de connexions et, pour ça, merci la [loi des grands nombres](https://fr.wikipedia.org/wiki/Loi_des_grands_nombres).

## Endpoints  

Kubernetes maintient, dynamiquement, pour chaque `Service`, une liste des cibles vers lequelles il va loadbalancer c'est à dire la liste des **pods ready correspondants au selector choisi** (en gros `kubectl get pods -l label=value` avec ready `n/n`). C'est ce qu'on appelle les `Endpoints`.  

```sh
gon@laboitemagique:~$ kubectl get endpoints
NAME            ENDPOINTS                                 AGE
myapp           10.233.112.205:9898,10.233.122.235:9898   20m
```  

Une ligne par `Service`. Si la colonne `ENDPOINTS` est vide c'est qu'il n'y a actuellement aucun **pod ready correspondant au selector choisi** donc que soit tous vos replicas ont crashé, soit le selector (ou les labels des pods) n'est pas bon.

## DNS interne

On a maintenant une IP stable comme point d'entrée mais, bien que les IP ça soit rigolo, c'est bien plus pratique de manipuler des noms d'hôte.  
Bonne nouvelle, Kubernetes intègre un serveur DNS interne dont le rôle est d'attribuer et résoudre des noms pour chaque `Service`. Il explique différents fournisseurs DNS compatibles Kubernetes (tout comme il existait plusieurs providers CNI) mais le plus fréquemment utilisé est probablement [coredns](https://github.com/coredns/coredns).  

**Chaque service hérite automatiquement d'un nom DNS interne de la forme `nomduservice.namespace.svc.cluster.local`**.  
Vous pouvez donc, depuis un pod du cluster, utiliser ce nom DNS plutôt que l'IP du service.  

> [!NOTE]  
> Bien que le nom DNS complet du service soit `nomduservice.namespace.svc.cluster.local` il est possible d'omettre les parties implicites. Par exemple `nomduservice.namespace` ou même, au sein du même namespace : `nomduservice`.

```sh
gon@laboitemagique:~$ kubectl exec -it olivier-6fd84b8877-stw85 -- sh
~ $ curl http://myapp:80
{
  "hostname": "olivier-6fd84b8877-stw85",
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

C'est la bonne façon d'exposer en intra-cluster et de contacter une application.  

## Autres types de services

> [!NOTE]  
> Dans tout ce chapitre, on a utilisé des `Service` de type `ClusterIP`. C'est le type par défaut, qui permet d'exposer en intra-cluster en obtenant une IP interne au cluster (d'où `ClusterIP`).  
Il existe 2 autres types de `Service` : `NodePort` et `LoadBalancer` qui apportent en plus des fonctionnalités d'exposition extra-cluster qui seront utilisées dans le chapitre suivant.