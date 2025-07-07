# Obtenir un cluster Kubernetes

Il existe différentes façons d'obtenir un cluster Kubernetes.  
Du cluster mononoeud sur son pc portable au cluster multi-noeuds physiques on-premise en passant par un cluster de VM managé chez un cloud provider il existe une multitude de façon d'obtenir un cluster Kubernetes.

## Prérequis

- [Kesako Kubernetes](../kesako/)

## Ca veut dire quoi installer Kubernetes ?

On l'a vu, Kubernetes est un orchestrateur de conteneurs.  
Quand on parle d'installer Kubernetes, on parle d'installer toute l'architecture(cf [Kesako Kubernetes](../kesako/)), faire en sorte que tout le monde communique bien ensemble.  
On considère qu'un cluster Kubernetes est opérationnel quand on peut discuter avec son APIServer (le webservice qui permet d'interagir avec le cluster, il sera étudié plus en détail dans le chapitre suivant).

## On-premise vs cloud

Kubernetes est une technologie utilisable à la fois à la maison (le concept de maison étant variable de votre pc portable jusqu'à votre datacenter) ou en location chez des fournisseurs de Cloud (GCP, AWS, OVH ...).  
Dans le premier cas on parle d'installation `on-premise`, dans le second on parle de `cloud`.  
A noter : on parle souvent de `cloud-native` pour qualifier les technologies modernes dont Kubernetes. Le fait d'être une technologie `cloud-native` ne veut pas dire qu'il s'agit d'une technologie disponible uniquement chez un cloud provider, c'est un faux ami !  
Vocabulaire : les offres Kubernetes des cloud providers sont en général "clé en main", le cloud provider s'occupe d'installer et configurer l'installation de base du cluster. Les `controlplanes` et les `etcd` sont mutualisés et optimisés, seuls les workers sont "visibles". Le cloud provider facture en général uniquement la puissance des workers et, pour certains, un frais fixe de gestion. On parle de cluster `managé`.

## On-premise : un cluster à la maison

Kubernetes est un projet opensource regroupant de nombreux composants (controller, scheduler, APIServer, kubelet ...) qui intéragissent ensemble. Bien qu'il soit possible d'installer et configurer manuellement chacun de ces composants, il est général beaucoup plus simple et pratique d'utiliser une distribution Kubernetes.  
Un peu à l'instar des distributions `linux`, il existe de nombreuses distributions Kubernetes différentes, avec des licenses éventuellement différentes (y compris des distributions payantes), des périmètres plus ou moins étendus (par exemple certaines distributions sont très simples, d'autres pré-installent de nombreuses applications en plus), des procédures d'installation différentes ... Le choix de la bonne distribution n'est pas simple mais, globalement, une fois installées toutes les distributions se comportent de la même façon et la compatibilité entre les distributions est en général très bonne (un cluster Kubernetes est un cluster Kubernetes).

### Exemples de différentes distributions

**[Docker desktop](https://docs.docker.com/desktop/kubernetes/)** : Kubernetes est inclu dans les versions récentes de Docker desktop. Cette distribution n'est cependant adaptée que pour du développement ou du testing, pas pour de la production `The Kubernetes server runs locally within your Docker instance, is not configurable, and is a single-node cluster. It runs within a Docker container on your local system, and is only for local testing.`.  
**[Minikube](https://minikube.sigs.k8s.io/docs/start/)** : Une distribution locale mono-noeud légère.  
**[k3s](https://k3s.io/)** : Une distribution légère supportant le multi noeud. Quelques limitations mais, contrairement aux distributions précédentes, on commence à pouvoir envisager de la prod.    
**[kubespray](https://github.com/kubernetes-sigs/kubespray)** : Une distribution multi-noeuds, production-ready. Cette distribution est basée sur [ansible](https://www.ansible.com/).  
**[openshift](https://www.redhat.com/en/technologies/cloud-computing/openshift)** : Une distribution mutli-noeuds, production-ready, payante (il existe une version community gratuite, libre mais peu utilisée : https://www.okd.io/). Openshift possède de nombreuses extensions à Kubernetes (ex : le client `oc`, le concept de `routes`, une politique de sécurité plus aggressive ...) ce qui la fait diverger du Kubernetes "standard"

Une fois le cluster installé, la distribution génère en général un fichier `kubeconfig` contenant les informations de connexion à l'APIServer.

## Cloud : un cluster en location

Tous les grands cloud providers ont une offre Kubernetes managée : `GKE` pour `GCP`, `EKS` pour `AWS`, `AKS` pour `azure` ...  
La création d'un cluster managé prend en général environ 5 minutes. Une fois le cluster créé, le cloud provider met à disposition en téléchargement (ou via la `CLI` du cloud provider) le fichier `kubeconfig`.  