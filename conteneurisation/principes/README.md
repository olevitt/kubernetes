# Conteneurisation

## Enjeux  

L'enjeu principal de la conteneurisation est d'**isoler les processus** qui tournent sur un même système / sur une même machine.  
Parvenir à **isoler** chaque processus **efficacement** permet de faire **cohabiter** des processus éventuellement très hétérogènes sans risque de **compatibilité**, **sécurité** et d'impact sur les **performances**.  
Plus le surcoût d'isolation est faible, plus la **densité du système d'information** pourra être grande (i.e : on utilise efficacement les ressources matérielles à disposition).  
  * La cohabitation native directement sur un système d'exploitation classique (que ça soit Windows ou un système Linux) n'apporte pas suffisamment de garanties d'isolation, que ça soit sur la **compatibilité** (exemple : faire cohabiter plusieurs versions d'un même logiciel), sur la **sécurité** (les applicatifs ont par défaut accès à l'ensemble du système de fichiers, pas d'isolation réseau ...) ou sur les **performances** (les processus ne sont pas limités en CPU ou en mémoire et sont en compétition directe les uns avec les autres).  
  * Les machines virtuelles (VM) ont révolutionné l'isolation en proposant d'isoler directement des sous-systèmes d'exploitation. **L'isolation se fait donc au niveau de l'OS entier**. L'adoption des VM a été massive et la plupart des systèmes d'information actuels sont basés sur des machines virtuelles. Le problème de l'approche d'isolation au niveau OS est qu'elle est couteuse, chaque **VM** ayant un **surcoût** à la fois en ressources (CPU, mémoire) et en gestion (montées de version, gestion de l'état ...).  

La conteneurisation propose une isolation au niveau **processus** afin de limiter le surcoût et la gestion.  
Le nom fait écho directement à la [révolution des conteneurs dans le transport maritime](https://www.youtube.com/watch?v=AXqm_r5apls)

## C'est quoi ?

La conteneurisation est un ensemble de techniques visant à faire tourner un processus dans un environnement virtuel, isolé du système hôte et des autres processus.

Le moteur de conteneurisation le plus connu est [`Docker`](https://www.docker.com/) mais il en existe d'autres (`containerd`, `lxc` ...). Un abus de langage courant est de confondre la technologie (`Conteneurisation`) et une implémentation (`Docker`).

## Conteneur ou machine virtuelle ?

![](img/vm.png)
_Source : https://www.docker.com/resources/what-container_

Les machines virtuelles utilisent une isolation au niveau système. Cette virtualisation conduit à la création et à la vie d'un OS (Operating System) invité par machine virtuelle.

![](img/conteneurs.png)
_Source : https://www.docker.com/resources/what-container_

La conteneurisation est une isolation applicative et logique. Il n'y a qu'un seul système.

## Image et conteneur

Il y a deux concepts importants à ne pas mélanger : `image` et `conteneur`.  
Une `image` correspond à la définition de l'environnement.  
Un `conteneur` correspond à un processus en cours d'exécution. Ce service a comme état initial celui défini dans une `image` mais vit ensuite sa vie (et sa mort).

Une même `image` peut donc être à l'origine de nombreux `conteneur`s. On publie des images et on fait tourner des conteneurs.

## Avantages de la conteneurisation

La conteneurisation propose 2 avantages majeurs :

- Légèreté : chaque conteneur ne vient qu'avec ce dont il a strictement besoin. Il n'y a donc pas de multiplication des OS. Chaque conteneur se lance donc beaucoup plus rapidement et consomme moins de ressources (CPU / mémoire). Il est donc possible de déployer, sur une même machine, beaucoup plus de conteneurs que l'équivalent en machines virtuelles. On parle de `densification`.
- Standardisation des livrables et process : le concept d'image est agnostique de la technologie sous-jacente (on peut faire cohabiter des conteneurs `python`, `Java`, `node` ... sans se poser de questions) et garanti le même comportement quel que soit l'environnement d'exécution.

## Comment ça marche ?

La conteneurisation s'appuie sur des technologies du noyau linux dont les `namespaces` et les `cgroups`.  
Un aperçu des différentes technologies sous-jacentes est disponible ici : https://medium.com/@BeNitinAgarwal/understanding-the-docker-internals-7ccb052ce9fe

## Aller plus loin

[![Une bonne vidéo](https://img.youtube.com/vi/0qotVMX-J5s/maxresdefault.jpg)](https://www.youtube.com/embed/0qotVMX-J5s)

La suite logique à ce tutoriel est de prendre en main Docker : [Hello docker](../../docker/hello).
