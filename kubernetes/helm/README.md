# Helm : packaging  
  
Pour intéragir avec Kubernetes, par exemple pour déployer une application, on est amenés à écrire de nombreux fichiers `.yaml` décrivant la stack complète.  
La gestion de ces fichiers yaml peut être fastidieuse.  
Helm vise à simplifier cette gestion en proposant des paquets (appelés `charts`) réutilisables.  

## Principe de Helm  

L'idée de Helm est de séparer ce qui relève de la structure des objets de ce qui relève de leur configuration spécifique pour une installation précise.    
Prenons l'exemple de l'Ingress d'une application web classique :  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: myapp
            port: 
              number: 8080
```  

Cet ingress mélange à la fois la structure d'un ingress classique et les valeurs spécifiques à cette installation (`host: myapp.example.com`, `service.name: myapp`, `number: 8080` ...).  
Le principe de Helm est de créer des paquets réutilisables dont les valeurs spécifiques sont variabilisées pour pouvoir être modifiées pour chaque installation. On parle de `templates`, c'est à dire des modèles.  
Ainsi, pour le chart réutilisable de cette application, on pourrait avoir un template un peu comme ça (version simplifiée) :  

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: {{ .Values.service.name }}
            port: 
              number: {{ .Values.service.port }}
```  
On pourra alors installer directement cette application en renseignant les valeurs spécifiques à notre installation (`values.yaml`) :  

```yaml
ingress:
  host: myapp.example.com
service:
  name: myapp
  port: 8080  
```  

En combinant le chart et les valeurs spécifiques à une installation, Helm est capable de générer les manifests Kubernetes exacts nécessaires pour installer l'application avec la bonne configuration.   

## Installation de Helm  

Helm, comme `kubectl`, est un binaire standalone donc s'installe sans dépendance : https://helm.sh/docs/intro/install/#from-the-binary-releases  

## Configuration de Helm  

Helm n'a pas besoin d'être configuré spécifiquement. Helm va automatiquement utilisé le même `kubeconfig` que `kubectl`.  
**Si kubectl est bien configuré, Helm est bien configuré**  

> [!NOTE]  
> Vous pouvez quand même utiliser Helm sans avoir accès à un cluster Kubernetes mais vous serez limités aux fonctionnalités de génération de templates `helm template`, vous n'aurez évidemment pas accès aux fonctionnalités d'installation (`helm install`)

## Trouver le bon chart  

Helm, encore plus que Docker, repose sur un système de registres décentralisé. Ainsi, n'importe qui peut publier et héberger des charts. On verra plus tard comment créer son propre chart et le publier.  
On peut citer les sources suivantes :  
  * [Bitnami](https://github.com/bitnami/charts) : des charts très complets pour des dizaines d'applications.  
  * [Dockerhub](https://hub.docker.com/) : principalement connu pour héberger des images Docker, Dockerhub propose aussi d'héberger des charts Helm  
  * Chaque projet publie et héberge en général les charts correspondant à son application. Par exemple, [ingress-nginx](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx), [argoCD](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd), [Onyxia](https://github.com/InseeFrLab/onyxia/tree/main/helm-chart) ...   

## Hello Helm  

On se propose d'installer Wordpress, le célèbre [CMS](https://fr.wikipedia.org/wiki/Syst%C3%A8me_de_gestion_de_contenu).  
Plutôt que de construire nous même les différents fichiers `yaml` nécessaires (Deployment, Service, Ingress, PVC ...), on va réutiliser le paquet Wordpress de Bitnami et se contenter de le configurer en fonction de nos besoins.  

L'ensemble des valeurs configurables sont définies dans le fichier [values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/wordpress/values.yaml). Oui, il y en a beaucoup ! Mais, pas de panique, l'immense majorité des valeurs définies par défaut sont satisfaisantes.  

D'ailleurs, on peut générer les contrats correspondants à une installation par défaut (sans surcharge de la configuration) : `helm template oci://registry-1.docker.io/bitnamicharts/wordpress`. Pas mal, non ?  

Prenez le temps d'analyser les manifests générés par Helm. Si vous en êtes satisfaits, vous pouvez passer à l'installation :  
`helm install wordpress-olivier oci://registry-1.docker.io/bitnamicharts/wordpress`.  

> [!NOTE]  
> `helm install` correspond en fait à la génération des manifests (`helm template`) + leur application dans le cluster (`kubectl apply` de ces manifests). Helm garde en plus (via des `Secrets`) une trace de ce qui a été installé ce qui facilite les mises à jour et les désinstallations  

On peut constater que le wordpress a bien été installé (avec d'ailleurs un mariadb) : `kubectl get pods`. Puissant, non ?  
Par contre, on peut constater qu'il n'y a pas d'Ingress. C'est normal, `ingress.enabled: false` par défaut dans [le fichier de values](https://github.com/bitnami/charts/blob/ded08d12fe99a5ed37d87af89dc5da2f23226ecd/bitnami/wordpress/values.yaml#L622)  
On va donc maintenant activer l'Ingress en surchargeant la configuration.  
Pour ça, on va créer un fichier de values (e.g : `values.yaml`) spécifique à notre installation :  
```yaml  
ingress:
  enabled: true
  hostname: monwordpress.example.com
```
**Il n'est pas nécessaire de répeter les valeurs que l'on ne veut pas surcharger**. Notre fichier de values spécifique à cette installation ne contient donc que les valeurs surchargées.  
Plutôt que de désinstaller notre wordpress (`helm uninstall`) pour le réinstaller à nouveau, on va utiliser la commande `helm upgrade` afin de ne modifier que ce qui a changé (exactement comme un `kubectl apply`).  

`helm upgrade wordpress-olivier oci://registry-1.docker.io/bitnamicharts/wordpress -f values.yaml`  
L'option `-f values.yaml` permet de prendre en compte nos valeurs surchargées. Cette option est disponible pour les commandes `helm install`, `helm upgrade` mais aussi `helm template`.  

On peut constater que l'Ingress est maintenant disponible ! `kubectl get ingress`.  

### Faire le ménage  

Pour lister les applications `Helm` actuellement installées (`release`) : `helm ls`  
Pour désinstaller une application (équivalent de `kubectl delete` de l'ensemble des ressources générées) : `helm uninstall wordpress-olivier`  

## Créer un chart  

Pour créer un chart, on peut soit repartir d'un chart existant en le dupliquant, soit repartir de 0 en utilisant par exemple `helm create monchart`.  
La structure est la suivante :  
  * `Chart.yaml` : métadonnées (nom, version, dépendances ...) du chart  
  * `templates` : dossier contenant les templates à installer  
  * `values.yaml` : ensemble des valeurs surchargeables avec leurs valeurs par défaut  

## Tester un chart local  

Pour tester (ou installer) un chart local, il est possible de référencer le dossier contenant ce chart dans les commandes install, upgrade ou template.  
Par exemple :  

```
helm create monchart  
helm template monchart
```

## Publier un chart  

[Tuto](https://forums.docker.com/t/how-to-build-push-helm-chart-to-docker-hub/133548)  




