# Késako Kubernetes ?

## Prérequis

- [Principes de la conteneurisation](../../conteneurisation/principes/README.md)
- [Hello Docker](../../docker/hello/README.md)

## Orchestrateur de conteneurs

On a vu que la conteneurisation permettait d'isoler des processus afin qu'ils cohabitent au sein d'un système hôte.  
Un système d'information étant constitué de multiples machines (physiques et/ou virtuelles), il est nécessaire d'industrialiser la conteneurisation afin de passer d'une technologie mono-machine ("je fais tourner des conteneurs sur une machine avec Docker") à une technologie multi-machine ("j'orchestre des conteneurs au sein de clusters de machines"). Un orchestrateur de conteneurs répond à ce besoin. Il est en charge de la répartition des conteneurs sur les différentes machines du cluster, du suivi de leur état, de la re-répartition en cas de panne ...

**La conteneurisation est la technologie au niveau micro (sur une machine), l'orchestration de conteneurs est la technologie au niveau macro (sur un cluster de machines)**

A noter : on entend régulièrement parler de Docker compose. C'est un outil d'orchestration locale (sur une machine) de conteneurs. C'est un espèce de milieu entre les niveaux micro et macro précedents. La démocratisation des orchestrateurs de conteneurs tend à éclipser de plus en plus Docker compose qui se retrouve mal positionné (pourquoi investir sur une technologie de plus pour faire de l'orchestration limitée à un seul noeud si on peut de plus en plus facilement faire de l'orchestration multinoeuds ?)

## Kubernetes

Kubernetes est un orchestrateur de conteneurs. Bien qu'il en existe / existait d'autres (par exemple [Mesos](https://mesos.apache.org/)), il est clair que Kubernetes a gagné la guerre.  
Originellement un projet interne de Google, [Kubernetes](https://github.com/kubernetes/kubernetes) a été opensourcé en 2014.

## Architecture

![](img/architecture.png)

Un cluster kubernetes = un ensemble (modifiable dynamiquement en ajoutant / enlevant des machines) de machines (physiques ou virtuelles) avec 3 rôles :

**Worker** : les muscles du cluster. C'est eux qui font tourner les applications (appelées pods) qui correspondent à des conteneurs. C'est eux qui nécessitent le plus de puissance. Renforcer le cluster = rajouter des workers.  
**Controlplane** (anciennement appelé `master`) : les "cerveaux" du cluster, ils reçoivent les ordres (par exemple : déployer une nouvelle application, récupérer la liste de tout ce qui tourne ...) et répartissent le boulot entre les workers. C'est eux qui portent l'APIServer, le webservice qui permet d'interagir avec le cluster.  
**etcd** : les bases de données des controlplanes. Ils stockent les informations sur l'état du cluster. Perdre les etcd = perdre l'état du cluster.

Chaque machine peut accomplir un ou plusieurs des rôles précédents.  
Pour des expérimentations, il est même possible de faire porter les 3 rôles à une unique machine (cluster mono noeud).  
Pour de la production, il est conseillé de séparer les controlplanes des workers (sécurité et performance). Controlplanes et etcd sont couramment sur les même machines.

Les `controlplanes` et les `etcd` sont des sytèmes distribués c'est à dire que chaque instance discute avec les autres instances pour s'échanger l'information et qu'il est possible à n'importe quel moment d'interroger n'importe quel `controlplane` ou n'importe quel `etcd`, le résultat sera le même peu importe quelle instance répond.  
Le fait d'avoir plusieurs instances de chaque permet d'assurer une meilleure résilience du cluster en permettant de survivre à une ou plusieurs pannes.  
Contrairement aux `controlplanes` qui sont stateless (sans état, les données sont dans l'etcd), les `etcd` ont un état. Il faut donc bien garder en tête un principe des systèmes distribués à état : la cohérence est obtenue via un [consensus](<https://en.wikipedia.org/wiki/Consensus_(computer_science)>). Ce système permet une tolérance aux pannes (`fault tolerance`) de `(n-1)/2`, arrondi à l'inférieur. Ce qui veut dire que pour être résilient à la perte d'un noeud, il en faut au moins 3. Pour une tolérance à une perte 2 noeuds, il faut monter à 5 noeuds et ainsi de suite. Funfact : passer de 1 à 2 noeuds n'augmente pas la tolérance aux pannes (0) mais augmente le risque de pannes. Un système distribué à 2 noeuds est donc moins stable qu'un système mononoeud !

Nombre de noeuds :

- Workers : entre 0 et +infini. Un cluster Kubernetes peut "fonctionner" sans aucun worker, il sera capable de prendre en compte des demandes mais ne pourra exécuter aucune charge de travail, tout sera mis en attente jusqu'à l'apparition de noeuds worker
- etcd : 1,3 ou 5. On choisi un nombre impair (cf point précédent), en général 3 ou, quand c'est possible 5. Bien qu'il soit possible d'en avoir plus, c'est en général ni utile ni rentable.  
- Controlplanes : au moins 1. Vivre avec 1 seul controlplane est risqué car s'il tombe, on n'a plus de moyen d'intéragir avec le cluster. 


## Aller plus loin

[![Watch the video](https://img.youtube.com/vi/aSrqRSk43lY/maxresdefault.jpg)](https://www.youtube.com/embed/aSrqRSk43lY)

La suite logique à ce tutoriel est l'obtention d'un cluster Kubernetes : [Mon premier cluster Kubernetes](../premiercluster).
