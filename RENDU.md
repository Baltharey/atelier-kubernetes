# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** Balthazar REY
**Groupe :** RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds `Ready`. `k3d-lab-server-0` est le control plane : il fait tourner l'API Server, le scheduler, le controller-manager et stocke l'état du cluster. `k3d-lab-agent-0` et `k3d-lab-agent-1` sont les workers : ils exécutent les pods (kubelet + containerd).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > 4 conteneurs : les 3 nœuds (image `rancher/k3s`) et `k3d-lab-serverlb`, le load balancer qui relie le port 8081 à Traefik. Un nœud k3d est un conteneur Docker qui fait tourner k3s ; les pods tournent à l'intérieur, d'où leur absence dans `docker ps`. Sur le cluster vide, `kube-system` contient déjà coredns, traefik, metrics-server, local-path-provisioner et svclb-traefik.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un Deployment `web`, qui a créé un ReplicaSet, qui a créé le Pod. `kubectl expose` a ensuite ajouté le Service `web` (ClusterIP).
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Un nouveau pod est apparu aussitôt, sous un autre nom (`web-68d995574f-h6xgl` → `web-68d995574f-wvsdw`). Le ReplicaSet doit maintenir 1 réplica : il compare l'état désiré à l'état réel et recrée le pod manquant (boucle de réconciliation).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non : le nouveau pod repart de l'image nginx d'origine. Le système de fichiers d'un conteneur est éphémère ; pour garder un contenu, il faut le décrire hors du conteneur (ConfigMap, volume ou image).
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > `port-forward svc/web` ouvre un tunnel vers un seul pod, choisi au lancement. Quand ce pod disparaît, le tunnel casse (`lost connection to pod`) et ne bascule pas vers le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `create` est impératif : il crée l'objet et échoue s'il existe déjà. `apply` est déclaratif : il crée ou met à jour l'objet pour qu'il corresponde au fichier, et peut être relancé sans risque.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Il désigne les pods qui reçoivent le trafic (ici `app: site`), dont Kubernetes tient la liste (les endpoints). S'il ne correspond à aucun pod, le Service n'a aucun endpoint et les connexions échouent.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
   site-565df9cf65-4kf9d   1/1     Running   0          25s   10.42.0.13   k3d-lab-agent-1    <none>           <none>
   site-565df9cf65-8624n   1/1     Running   0          33s   10.42.2.16   k3d-lab-agent-0    <none>           <none>
   site-565df9cf65-fppm2   1/1     Running   0          22s   10.42.1.10   k3d-lab-server-0   <none>           <none>
   ```
   > Un pod par nœud : le scheduler répartit les réplicas, et avec k3s le nœud server accepte aussi des pods.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > `ErrImagePull`, puis `ImagePullBackOff`. La cause est dans la section *Events* de `kubectl describe pod <pod>` : `docker.io/library/nginx:9.99: not found`.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui. Le rolling update crée d'abord un nouveau pod et attend qu'il soit prêt avant de supprimer un ancien. Le pod `9.99` ne démarrant jamais, les 3 pods `nginx:1.28` ont continué de servir. Après `rollout undo`, `kubectl diff` ne montrait plus d'écart avec les fichiers.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://humble-giggle-7v75v65vjwphr5xr-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui, à l'API Server : `kubectl` depuis le Codespace avec le kubeconfig, Portainer depuis l'intérieur du cluster avec son ServiceAccount. `mario` est un Deployment standard ; créé avec `kubectl`, il apparaît comme « External » dans Portainer.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > - **A — Ingress** : Traefik envoie le port 8081 vers le Service `site`. Pendant la suppression d'un pod, 10 requêtes sur 10 ont réussi, contrairement au `port-forward`.
    > - **B — Probes et ressources** : `readinessProbe` sur `/` et requests/limits ajoutés. Avec `/nope`, le pod reste `0/1` (erreur 404) : il est retiré des endpoints et ne reçoit plus de trafic, sans être redémarré.
    > - **C — Drain** : les pods d'`agent-0` ont été recréés sur les autres nœuds. Après `uncordon`, ils ne reviennent pas, car Kubernetes ne rééquilibre pas les pods en cours. Seul Portainer y est revenu, son volume local étant lié à ce nœud.
    > - **D — Namespaces** : une seconde copie du site tourne dans `prod`. Les mêmes noms y sont possibles, car les noms sont propres à chaque namespace.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18