# Mon premier déploiement

## Objectif

Maintenant que notre cluster est en place et que l'on peut communiquer avec l'APIServer (`kubectl get nodes`), on va pouvoir déployer notre premier application !  
On se propose de déployer [Podinfo](https://github.com/stefanprodan/podinfo), un serveur web très simple affichant une unique page.  
A noter : il est tout à fait possible de remplacer Podinfo par n'importe quel autre image Docker dans toute la suite. Par exemple pourquoi ne pas utiliser l'image de [Ma première image docker](../../docker/creer-image/)

## Préparatifs

### Environnement

En plus d'avoir besoin d'un `kubectl` bien configuré, il est recommandé d'utiliser [vscode](https://code.visualstudio.com/) ainsi que son [plugin kubernetes](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) afin de bénéficier de l'autocomplétion, d'un linter et de tout un tas d'autres outils offrant un gain de temps non négligeable.  
Les IA comme `chatGPT` sont très performantes dans l'écriture et le débug de contrats Kubernetes.

### Test local de l'image

Bien que ce ne soit pas strictement nécessaire, il est en général bienvenu de tester localement l'image avant de la déployer dans Kubernetes. En particulier, il convient de déterminer le nom de l'image, son port d'écoute (le cas échéant), ses éventuels paramètres de configuration (en particulier les variables d'environnement) et les éventuels chemins à monter en volumes.  
La [documentation de PodInfo](https://github.com/stefanprodan/podinfo#docker) donne toutes ces informations en une seule ligne :

```
docker run -p 9898:9898 stefanprodan/podinfo
```

Nom de l'image : `stefanprodan/podinfo`  
Port d'écoute : `9898`  
Variables d'environnements / volumes : pas nécessaire

L'ouverture de `localhost:9898` dans un navigateur devrait afficher la page de PodInfo.

Afin de limiter le risque d'erreurs et de faciliter le débug il est important, surtout dans le cadre d'architectures comme celles là qui contiennent de nombreuses briques, de tester unitairement chaque couche avant de passer à la suivante. Le test local de l'image permet de confirmer que l'image est bonne avant de passer à son orchestration dans le cluster.

## Manifests

Pour déployer quelque chose dans Kubernetes, il faut envoyer des contrats (aussi appelés manifests) à l'APIServer. Si la syntaxe est bonne et que les droits sont suffisants, l'APIServer enregistrera ce nouveau contrat dans la base (`etcd`) et le controller fera en sorte de toujours faire le maximum pour atteindre l'état souhaité. L'état réel peut diverger, au moins temporairement, de l'état souhaité si par exemple le cluster manque de ressources sur les workers. Le controller va dédier son existence à faire converger l'état du cluster vers l'état souhaité (ensemble des manifests).  
Ces contrats sont écrits en [YAML](https://yaml.org/) et sont en général appliqués via `kubectl apply -f moncontrat.yaml`.

## Pod

L'objectif de Kubernetes est d'orchestrer des conteneurs qui vont tourner sur les conteneur engines des noeuds workers.  
L'élément de base de Kubernetes est le `Pod`, littéralement une `gousse` de conteneurs.  
Tout conteneur orchestré par Kubernetes appartient à un `Pod`. Un `Pod` peut potentiellement contenir plusieurs conteneurs qui partageront un contexte commun mais, pour simplifier, on va commencer par considérer que `1 pod = 1 conteneur`.

## Deployment

On pourrait demander à Kubernetes la création d'un `Pod` pour lancer notre application mais, si pour une raison quelconque il arrivait malheur à ce pod (le processus du conteneur se termine, il est tué par manque de mémoire, il crash ...) ce serait fini pour notre application.  
Ainsi, on préfère en général, en particulier pour les applications longue durée comme un serveur web, déployer un objet qui va controler nos pods et s'assurer qu'il y en a toujours le nombre demandé en vie. On parle de `controllers`. Le plus connu des `controllers` est le `deployment`.  
Un `deployment` coorrespond à une demande d'avoir en permanence `n` pods vivants.

Le plugin `Kubernetes` pour vscode facilite l'écriture des contrats.  
Ainsi, pour écrire un contrat de `deployment`, il suffit de créer un nouveau fichier `.yaml` et de taper les premières lettres de l'objet à définir `Deplo`. L'autocomplétion va automatiquement générer un premier contrat :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: <Image>
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: <Port>
```

`kind: Deployment` : il s'agit bien d'un objet `Deployment`. Chaque concept en Kubernetes correspond à un objet, représenté par un certain `Kind`. Il en existe plusieurs centaines de base dans Kubernetes et il est possible d'en définir de nouveaux au travers d'extensions (`CRD` : custom resource definition).  
`apiVersion: apps/v1` : l'apiVersion permet à Kubernetes de publier des nouvelles définition de certains `Kind` tout en gardant la compatibilité avec les définitions précédentes. Impossible d'inventer la valeur à mettre, on cherche sur google si le plugin ne fait pas le travail pour nous.

```
metadata:
  name: myapp
```

La métadonnée `name` correspond au nom de la ressource créée, elle joue le rôle d'identifiant. Le trio d'unicité `Kind`, `name`, `namespace` (espace de travail, on en parlera plus tard).

```yaml
selector:
  matchLabels:
    app: myapp
template:
  metadata:
    labels:
      app: myapp
```

On verra plus tard, promis.

```yaml
spec:
  containers:
    - name: myapp
      image: <Image>
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
      ports:
        - containerPort: <Port>
```

Notre pod sera constitué d'un unique conteneur dont l'image est `<Image>`.  
Il aura des limites de ressources à 128Mo de ram, 0.5CPU (ne peut pas consommer plus d'un demi CPU par cycle).  
Il exposera des choses sur son port `<Port>`.

Après modification, on obtient le contrat suivant :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: stefanprodan/podinfo
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 9898
```

On peut maintenant appliquer le contrat : `kubectl apply -f deployment.yaml`.  
Pour visualiser les deployments présents dans le cluster : `kubectl get deployment`.  
La présence du `deployment` est censée déclencher la création de pods. Vérifions : `kubectl get pods`. Il y a bien un pod qui tourne.

## Debug

Pour vérifier que tout va bien, on peut utiliser les commandes suivantes :  
`kubectl get pods` :

```sh
gon@laboitemagique:~$ kubectl get pods
NAME    READY   STATUS    RESTARTS       AGE
podname   1/1     Running   0  11m
```

Le pod podname est en bonne santé !

On peut accéder aux logs via la commande `kubectl logs podname`.  

## Nettoyage  

Pour nettoyer tout ce qui a été créé précedemment, on peut utiliser la commande `kubectl delete`.  
Soit en supprimant manuellement chaque ressource (`kubectl delete deployment myapp`) soit en référençant les fichiers qui ont servi à la création des ressources (`kubectl delete -f deployment.yaml`)

## Aller plus loin

La suite logique à ce tutoriel est le déploiement d'une première application sur le cluster : [Mon premier déploiement](../).
