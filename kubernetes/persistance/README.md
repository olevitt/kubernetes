# Persistance

Comme pour le niveau micro (cf [Docker, persistance : volumes](../../docker/avance/)), dès qu'un conteneur se termine sur Kubernetes, toutes les modifications du filesystem qui ont eu lieu sont perdues.  
Il va donc falloir, comme pour Docker, définir des volumes.  
A la différence de Docker, Kubernetes offre [plusieurs types de volume différents](https://kubernetes.io/docs/concepts/storage/volumes/) et pas simplement des montages du système hôte.  
Dans cette partie on va s'intéresser aux 2 types de volume principaux :   

    * HostPath : l'équivalent des montages du système hôte de Docker
    * PVC (`PersistentVolumeClaim`) : une demande d'espace de stockage qui suivra le pod partout où il ira

## Définir un volume 

Quel que soit le type de volume utilisé, on doit d'abord le définir au niveau du Pod avant de le monter.  

```yaml
    spec:
      containers:
        - [...]
      volumes:
        - name: monvolume
          hostPath:
            path: /var/logs
```

## Utiliser le volume  

Une fois le volume défini au niveau du Pod, on peut le monter dans les containers :  

```yaml
    spec:
      containers:
        - name: myapp
          image: postgres
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          hostPath:
            path: /var/data
```  

Le `name` du `volumeMounts` référence le `name` du `volumes`.  

## Volume HostPath  

Les volumes `HostPath` sont l'équivalent des montages Docker vu précédemment.  
On précise dans le volume quel dossier du système hôte on veut monter (dans l'exemple plus haut `/var/data`) et dans le `volumeMounts` dans quel chemin du conteneur on veut le monter.  

> [!CAUTION]
> Les volumes `HostPath` ne doivent être utilisés que de façon très très limitée car ils constituent directement une menace de sécurité. Les montages sont par défaut en lecture ET écriture. Imaginez ce qu'il se passe si vous laissez n'importe quel développeur monter, lire et écrire `/etc/passwd` de vos noeuds.  
Les volumes `HostPath` constituent aussi un frein à la portabilité de vos charges de travail. Si un pod utilise un `HostPath`, il va lire / écrire sur un noeud en particulier. Du coup il sera lié pour toujours à ce noeud ce qui est contraire au principe de faire un cluster dans lequel les charges de travail peuvent être schedulées sur n'importe quel noeud. Bref, comme pour `HostNetwork`, `HostPath` constitue une rupture de l'isolation qu'il faut réserver à des usages infra très précis.  

## PVC : PersistentVolumeClaim  

Les `PVC` offrent une façon flexible de réserver du stockage et de le monter dans les pods.  
Un `PVC` correspond à une demande de stockage, d'un certain type (`Storageclass`) et d'une certaine taille (`size`).  
Un `PVC` est ensuite honoré par un provisionner qui créé et lui attribue un `PV` (`PersistentVolume`) qui correspond au volume réel contenant les données.  
Pour qu'un `PVC` soit honoré, il faut qu'un provisionner soit configuré dans le cluster. C'est le cas des clusters managés par les cloud providers mais, pour les clusters on-premise, il faut en général en installer un (par exemple en utilisant [rook-ceph](https://rook.io/) ou [Longhorn](https://longhorn.io/)) pour que les PVC soient `bound`. Si aucun provisionner n'est disponible ou si la création de `PV` échoue (par exemple parce que le système de stockage est plein), le `PVC` va rester `Pending` jusqu'à ce que la situation se résorbe.  

### PVC : création d'un PVC  

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mon-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard 
```

`storageClassName` est optionnel. S'il n'est pas défini, la classe par défaut (si elle existe) sera utilisée.  
Certains clusters peuvent avoir plusieurs `storageClass` pour différencier le type de stockage (par exemple stockage rapide / lent, hdd vs ssd).  
Pour lister les `storageClass` disponibles : `kubectl get storageclass`.  

On applique ensuite le `PVC` : `kubectl apply -f pvc.yaml`  
On peut suivre son état (`Pending` => `Bound` une fois qu'un provisionner l'a honoré) : `kubectl get pvc`  

Une fois le `PVC` créé on peut le monter dans un conteneur du pod :  
```yaml
spec:
  containers:
    - name: postgres
      image: postgres
      volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-storage
  volumes:
    - name: postgres-storage
      persistentVolumeClaim:
        claimName: mon-pvc
```

Le `claimName` référence le nom du `PVC` préalablement créé
