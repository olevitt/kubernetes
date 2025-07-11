1. Définir, avec vos mots, la conteneurisation  
2. Définir, avec vos mots, Kubernetes  
3. Faire un schéma simple de l'architecture Kubernetes et expliquer succinctement le rôle de chaque composant  
4. Qu'est ce que `kubectl` ?  Comment le configure t'on ?
5. Expliquer le rôle d'un `Deployment`  
6. Expliquer le rôle d'un `Service` (ClusterIP)  
7. Expliquer le fonctionnement d'un `Ingress` et la différence entre un `Ingress` et un `Ingress controller`  
8. Que se passe t'il ici ?  
`kubectl get pods`  

| NAME                               | READY | STATUS             | RESTARTS | AGE   |
|------------------------------------|-------|---------------------|----------|-------|
| nginx-deployment-7c77b68cff-abcde | 1/1   | Running            | 0        | 3d4h  |
| api-backend-5fcb8c79d4-fghij       | 0/1   | CrashLoopBackOff   | 12       | 15m |

Que feriez vous pour continuer le débug ?  

9. Que se passe t'il ici ?  
`kubectl get pods`  

| NAME                               | READY | STATUS             | RESTARTS | AGE   |
|------------------------------------|-------|---------------------|----------|-------|
| frontend-app-76d8cb76db-klmno      | 0/1   | ImagePullBackOff   | 0        | 5m    |
| database-0                         | 1/1   | Running            | 0        | 7d    |

Que feriez vous pour continuer le débug ? 

10. Expliquer le principe de Helm et son intérêt
