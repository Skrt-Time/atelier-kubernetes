# Rendu — Atelier Kubernetes

**Nom / Prénom :** NONGA Arsène
**Groupe :** RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds. `k3d-lab-server-0` est le control plane : c'est le "chef", qui contient l'API Server et le scheduler et décide où lancer les pods. `k3d-lab-agent-0` et `k3d-lab-agent-1` sont les workers : ce sont eux qui font réellement tourner les conteneurs.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > On voit un conteneur Docker par nœud (server-0, agent-0, agent-1), plus `serverlb`, le load balancer qui expose le port 8081. Dans k3d, un nœud n'est donc pas une vraie machine : c'est un conteneur Docker qui fait tourner k3s, et nos pods tournent à l'intérieur.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ?
   > Un Deployment `web`, qui crée un ReplicaSet, qui crée lui-même le Pod. Le Service `web` vient de la commande `kubectl expose`.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Un nouveau pod, avec un autre nom, est apparu en quelques secondes. Le ReplicaSet vérifie en permanence qu'on a bien le nombre de pods demandé (1). Il en manquait un, donc il l'a recréé : c'est la boucle de réconciliation de Kubernetes.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non, on est revenu à la page « Welcome to nginx! ». Ce qu'on écrit dans un conteneur est temporaire : quand le pod est supprimé, ses modifications disparaissent avec lui, et le nouveau pod repart de l'image nginx d'origine. C'est pour ça qu'il faut mettre le contenu en dehors du conteneur (ConfigMap, exercice 4).

6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Le port-forward vise le Service, mais en réalité il se connecte à un seul pod. Quand ce pod est supprimé, le tunnel n'a plus de destination et s'arrête. Preuve :

Forwarding from 127.0.0.1:8090 -> 80
Handling connection for 8090
“An error occurred forwarding” err="error forwarding port 80 to pod b781ae55..., failed to find sandbox ... not found"
error: lost connection to pod
[4]+ Exit 1 kubectl port-forward svc/web 8090:80


## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `create` crée un objet une seule fois et renvoie une erreur s'il existe déjà. `apply` part d'un fichier YAML : il crée l'objet s'il n'existe pas ou le met à jour, et on peut le relancer autant de fois qu'on veut. Le fichier dans Git devient la référence.

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Il indique au Service vers quels pods envoyer le trafic, grâce aux labels (ici `app: site`). Si aucun pod n'a ce label, le Service existe mais n'a aucun endpoint : il ne répond donc pas.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ?

NAME READY STATUS RESTARTS AGE IP NODE
site-5f45f94d5c-kpfsz 1/1 Running 0 10m 10.42.1.8 k3d-lab-agent-1
site-5f45f94d5c-rp9lc 1/1 Running 0 10m 10.42.0.8 k3d-lab-agent-0
site-5f45f94d5c-znlkp 1/1 Running 0 11m 10.42.2.7 k3d-lab-server-0

> Kubernetes a mis un pod sur chaque nœud pour répartir la charge. Sur k3s, le server accepte aussi des pods, contrairement à un cluster classique.

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
 > Le pod est passé en `ErrImagePull` puis en `ImagePullBackOff`. La cause est donnée par `kubectl describe pod <nom-du-pod>`, dans la partie Events :
Warning  Failed   Failed to pull image "nginx:9.99": ... "docker.io/library/nginx:9.99": not found
Warning  Failed   Error: ErrImagePull
Warning  Failed   Error: ImagePullBackOff
> La version 9.99 n'existe pas sur Docker Hub. J'ai corrigé avec `kubectl rollout undo deploy/site`, puis vérifié avec `kubectl diff -f k8s/site/` que le cluster et mes fichiers étaient de nouveau identiques.

11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
> Oui. Pendant une mise à jour, Kubernetes remplace les pods un par un et ne supprime un ancien pod que lorsque le nouveau est prêt. Le nouveau pod n'a jamais démarré, donc les anciens ont continué à servir le site.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
> https://didactic-umbrella-g475qgqvvp43p7wg-8585.app.github.dev/

13. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
> Oui, les deux parlent à l'API Server. `kubectl get deploy mario` retrouve bien le Deployment créé depuis Portainer. Portainer n'est qu'une interface graphique qui envoie les mêmes requêtes que kubectl.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
> **A – Ingress** : Traefik envoie le trafic du port 8081 vers le Service `site`. Même en supprimant des pods, le site reste accessible, car l'Ingress suit le Service et pas un pod précis (contrairement au port-forward).
>
> **B – Probes et ressources** : j'ai ajouté une readinessProbe sur `/` et des limites CPU et mémoire. Si la probe pointe vers `/nope`, nginx répond 404 : les pods ne sont jamais "prêts" et ne reçoivent pas de trafic.
>
> **C – Drain** : `drain` vide le nœud agent-0, et ses pods sont recréés sur les autres nœuds. Après `uncordon`, ils ne reviennent pas tout seuls : Kubernetes ne déplace pas des pods qui fonctionnent déjà.
>
> **Problèmes rencontrés :**
> - J'ai collé plusieurs commandes après `kubectl exec -it` : elles se sont perdues, et un `exit` en trop a fermé mon terminal. Solution : `kubectl exec deploy/web -- sh -c '...'`, sans entrer dans le conteneur.
> - `kubectl apply` refusait mes fichiers (`containerPort of type int32`) : les TODO n'étaient pas enregistrés. J'ai enregistré les fichiers et vérifié avec `grep TODO`.
> - Le passage à 3 réplicas n'avait pas été pris en compte au premier essai, je l'ai corrigé.
> - Portainer s'est bloqué après 5 minutes, puis j'ai eu une erreur 403 sur le setup token. J'ai fait un `rollout restart` et récupéré le token avec `kubectl logs -n portainer deploy/portainer | grep setup_token`. Il fallait aussi sélectionner l'environnement `local` pour voir le menu Applications.
> - L'Ingress répondait 404 juste après l'`apply` : Traefik avait juste besoin de quelques secondes.
> - Le port 8080 était déjà utilisé par un autre tunnel, j'ai utilisé le port 8090 pour la preuve de l'exercice 3.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
