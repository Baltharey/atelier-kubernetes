# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** Balthazar REY
**Groupe :** RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > **3 nœuds**, tous `Ready` :
   > - `k3d-lab-server-0` (rôle `control-plane`) : le « master ». Il héberge l'**API Server** (point d'entrée unique de `kubectl`), le **scheduler** (choisit le nœud de chaque pod), le **controller-manager** (boucles de réconciliation : Deployments, ReplicaSets…) et la base qui stocke l'état du cluster. Avec k3s, il fait aussi tourner un kubelet : il peut donc lui aussi accueillir des pods.
   > - `k3d-lab-agent-0` et `k3d-lab-agent-1` (rôle `<none>`) : les **workers**. Ils exécutent les pods applicatifs grâce au **kubelet** (qui dialogue avec l'API Server) et au runtime **containerd**, et assurent le réseau des pods (flannel, kube-proxy).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre **4 conteneurs** : `k3d-lab-server-0`, `k3d-lab-agent-0` et `k3d-lab-agent-1` (image `rancher/k3s`), plus `k3d-lab-serverlb` (image `k3d-proxy`, un nginx qui sert de load balancer : il publie l'API sur un port de la machine et relie le port **8081** au port 80 des nœuds, donc à Traefik).
   > Concrètement, **un nœud k3d est un conteneur Docker** qui exécute k3s : chaque conteneur joue le rôle d'une machine du cluster, avec son propre kubelet et son propre containerd. Les pods tournent **à l'intérieur** de ces conteneurs (conteneurs dans des conteneurs) : c'est pour cela qu'on ne les voit pas dans `docker ps`.
   > Sur le cluster « vide », `kubectl get pods -A` montre déjà les composants système dans le namespace **`kube-system`** : `coredns` (DNS interne), `local-path-provisioner` (volumes), `metrics-server`, `traefik` (Ingress Controller), `svclb-traefik-*` (un par nœud, DaemonSet du load balancer de k3s) et les jobs terminés `helm-install-traefik*`.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > `kubectl create deployment web` crée un **Deployment** `web`, qui crée à son tour un **ReplicaSet** `web-<hash>`, qui crée le **Pod** `web-<hash>-<id>`. `kubectl expose` a ensuite ajouté le **Service** `web` (type ClusterIP, port 80). Le Service `kubernetes` affiché par `kubectl get all` existait déjà : c'est l'adresse interne de l'API Server.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Le pod est passé en `Terminating` et, presque en même temps, un **nouveau pod avec un nouveau nom** est apparu (`Pending` → `ContainerCreating` → `Running`).
   > Je n'avais pas créé un pod isolé mais un Deployment : son **ReplicaSet** doit maintenir 1 réplica. Il compare en permanence l'état désiré (1 pod) à l'état réel (0 pod) et recrée le pod manquant : c'est la **boucle de réconciliation** (auto-réparation). Le nouveau pod est un objet neuf, d'où le nouveau nom.
   <!-- auto:q4 -->
   >
   > Observé : le pod `web-68d995574f-h6xgl` a été remplacé immédiatement par `web-68d995574f-wvsdw` (nouveau nom).
   <!-- /auto:q4 -->

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > **Non** : après relance du tunnel, on retrouve la page « Welcome to nginx! ». La modification avait été faite dans la couche d'écriture du conteneur de l'ancien pod. Le **système de fichiers d'un conteneur est éphémère** : il disparaît avec le conteneur, et le nouveau pod repart de l'image `nginx` d'origine. Pour qu'un contenu survive, il faut le décrire en dehors du conteneur (image, ConfigMap, volume), ce que fait l'exercice 4.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > `kubectl port-forward svc/web` ne fait pas de répartition de charge : au lancement, il choisit **un seul pod** derrière le Service et ouvre un tunnel vers ce pod précis (via l'API Server et le kubelet). Quand ce pod est supprimé, la cible du tunnel n'existe plus : la connexion est perdue (`lost connection to pod` dans `/tmp/web.log`) et le processus s'arrête. Il ne bascule pas tout seul vers le nouveau pod : il faut le relancer.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : « crée cet objet ». Il échoue si l'objet existe déjà (`AlreadyExists`) et ne garde aucune trace de l'intention.
   > `kubectl apply` est **déclaratif** : « voici l'état désiré ». Il crée l'objet s'il n'existe pas, sinon il calcule la différence avec l'existant et n'applique que les changements (il mémorise la dernière configuration appliquée dans l'annotation `kubectl.kubernetes.io/last-applied-configuration`). On peut le relancer autant de fois qu'on veut (**idempotent**), et les fichiers YAML versionnés dans Git deviennent la référence.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` (ici `app: site`) dit **quels pods reçoivent le trafic** du Service : Kubernetes tient à jour la liste de leurs IP (les *endpoints*), en ne gardant que les pods prêts. C'est grâce à ces labels que le Service suit automatiquement les pods recréés ou ajoutés.
   > S'il ne correspond à aucun pod, le Service existe et a une ClusterIP, mais **n'a aucun endpoint** (`kubectl get endpoints site` affiche `<none>`) : les connexions échouent et le `port-forward` refuse de démarrer, faute de pod. (De même, le `selector` d'un Deployment doit correspondre aux labels de son `template`, sinon l'API refuse le manifeste.)

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   <!-- auto:q9 -->
   ```
   NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
   site-565df9cf65-4kf9d   1/1     Running   0          25s   10.42.0.13   k3d-lab-agent-1    <none>           <none>
   site-565df9cf65-8624n   1/1     Running   0          33s   10.42.2.16   k3d-lab-agent-0    <none>           <none>
   site-565df9cf65-fppm2   1/1     Running   0          22s   10.42.1.10   k3d-lab-server-0   <none>           <none>
   ```
   <!-- /auto:q9 -->
   > Le **scheduler** a réparti les 3 réplicas sur des nœuds différents : il évite par défaut de regrouper les pods d'un même Deployment, pour qu'une panne de nœud n'emporte pas tout le site. Avec k3s, le nœud `server-0` n'est pas réservé au control plane et peut, lui aussi, recevoir des pods.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod est resté en **`ErrImagePull`** puis **`ImagePullBackOff`** (`READY 0/1`) : Kubernetes ne parvient pas à télécharger l'image et réessaie avec un délai croissant. La cause exacte apparaît dans la section *Events* de **`kubectl describe pod <nom-du-pod>`** : `Failed to pull image "nginx:9.99"` (le tag `9.99` n'existe pas sur Docker Hub).
    <!-- auto:q10 -->
    >
    > Observé pendant la panne :
    > ```
    > NAME                    READY   STATUS         RESTARTS   AGE
    > site-5dbb754bd-pjfbr    0/1     ErrImagePull   0          7s
    > site-7d55b8d788-4twlb   1/1     Running        0          31s
    > site-7d55b8d788-ft6lh   1/1     Running        0          18s
    > site-7d55b8d788-lrs9w   1/1     Running        0          39s
    > Warning Failed 6s kubelet spec.containers{nginx}: Failed to pull image "nginx:9.99": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:9.99": failed to resolve reference "docker.io/library/nginx:9.99": docker.io/library/nginx:9.99: not found
    > Warning Failed 6s kubelet spec.containers{nginx}: Error: ErrImagePull
    > Normal BackOff 5s kubelet spec.containers{nginx}: Back-off pulling image "nginx:9.99"
    > Warning Failed 5s kubelet spec.containers{nginx}: Error: ImagePullBackOff
    > ```
    <!-- /auto:q10 -->
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > **Oui.** La stratégie **RollingUpdate** (par défaut `maxSurge: 25 %`, `maxUnavailable: 25 %`, soit 1 pod en plus et 0 pod en moins avec 3 réplicas) crée d'abord **un** nouveau pod et attend qu'il soit prêt avant de supprimer un ancien. Comme le pod `nginx:9.99` ne démarre jamais, la mise à jour reste bloquée et les **3 anciens pods (`nginx:1.28`) continuent de servir** ; le Service n'envoie du trafic qu'aux pods prêts. `kubectl rollout undo` a ensuite remis le modèle précédent, et `kubectl diff -f k8s/site/` n'affichait plus aucun écart : cluster et dépôt Git de nouveau alignés.
    <!-- auto:q11 -->
    >
    > Observé : site toujours en ligne pendant la panne : **oui**.
    <!-- /auto:q11 -->

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    <!-- auto:q12 -->
    > https://humble-giggle-7v75v65vjwphr5xr-8585.app.github.dev/
    <!-- /auto:q12 -->
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > `kubectl get deploy mario -o yaml` montre un Deployment `apps/v1` **standard**, de même nature que `web` et `site`. Seules quelques métadonnées diffèrent : Portainer ajoute ses propres labels (`io.portainer.kubernetes.application.*`) et apparaît comme gestionnaire dans les `managedFields`.
    > **Oui, ils parlent au même composant : l'API Server (`kube-apiserver`)** du control plane. `kubectl` s'y connecte depuis le Codespace avec le kubeconfig (contexte `k3d-lab`) ; Portainer s'y connecte depuis l'intérieur du cluster, via le Service `kubernetes` et son ServiceAccount doté des droits `cluster-admin`. L'API Server enregistre l'objet, puis le controller-manager et le scheduler font le reste, quel que soit l'outil qui a envoyé la demande.
    <!-- auto:q13 -->
    >
    > Observé : Mario a été déployé via **kubectl**. Labels du Deployment `mario` :
    > aucun label (déployé tel quel depuis k8s/mario/mario.yaml)
    <!-- /auto:q13 -->

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **A — Ingress (fait)** : `k8s/bonus/ingress.yaml` envoie `/` vers le Service `site` (port 80) via Traefik, joignable sur le port **8081**. En supprimant des pods, le site **reste accessible** : contrairement au `port-forward` lié à un seul pod, l'Ingress passe par le Service, qui suit en permanence la liste des pods prêts.
    >
    > **B — Probes et ressources (fait)** : `readinessProbe` HTTP sur `/` (port 80) et `requests`/`limits` (50m/100m de CPU, 32Mi/64Mi de mémoire) dans `deployment.yaml`. Avec une sonde sur **`/nope`**, le pod est `Running` mais **jamais prêt (`READY 0/1`)** : nginx renvoie 404, l'événement `Readiness probe failed: HTTP probe failed with statuscode: 404` se répète, le pod est **retiré des endpoints** du Service et ne reçoit aucun trafic. Il n'est pas redémarré (ce serait le rôle d'une `livenessProbe`), et un rolling update vers cette version resterait bloqué, avec les anciens pods qui continuent de servir.
    >
    > **C — Maintenance d'un nœud (fait)** : `drain` marque `k3d-lab-agent-0` `SchedulingDisabled` (*cordon*) puis **évince** ses pods ; leurs Deployments les recréent aussitôt sur `server-0` et `agent-1` (les pods de DaemonSet `svclb-traefik` sont ignorés). Après `uncordon`, le nœud accepte de nouveau des pods mais **les pods ne reviennent pas d'eux-mêmes** : le scheduler ne place que les *nouveaux* pods, et Kubernetes ne rééquilibre jamais des pods qui tournent. Pour redistribuer, il faut en recréer (`kubectl rollout restart deploy/site`). Seule exception possible : un pod resté `Pending` pendant le drain parce que son volume `local-path` est attaché à `agent-0` (cas de Portainer s'il y tournait) ; il y est replanifié dès le `uncordon`.
    >
    > **D — Namespaces (fait)** : namespace `prod` avec une seconde copie du site (`kubectl apply -n prod -f k8s/site/`). Les mêmes noms (`site`, `site-html`) coexistent car les noms sont propres à chaque namespace ; dans Portainer, le filtre par namespace montre les deux sites séparément. Cette isolation est **logique** (noms, droits, quotas) : sans NetworkPolicy, les pods des deux namespaces peuvent encore se joindre sur le réseau.
    <!-- auto:q14 -->
    >
    > **Traces de mon exécution :**
    > - A : pendant la suppression d'un pod, **10/10** requêtes sur le port 8081 ont réussi.
    > - B : avec `path: /nope` → `site-f58cfdbb6-6pvrk READY=0/1 STATUS=Running` ; événement : `Readiness probe failed: HTTP probe failed with statuscode: 404`
    > - C : pods évincés de `k3d-lab-agent-0` : `default/site-7d55b8d788-lrs9w kube-system/local-path-provisioner-5d9d9885bc-jw2d4 portainer/portainer-559cbdfc8b-twxd9 probe-demo/site-f58cfdbb6-6pvrk prod/site-54dbc4f754-jqdts` ; 15 s après `uncordon`, pods sur ce nœud (hors DaemonSet) : `portainer/portainer-559cbdfc8b-4khtn`.
    > - D : `default/site 3/3 · prod/site 3/3`
    <!-- /auto:q14 -->

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
