# Compte rendu TP PetitPanier — Mya / Chouchou

Systèmes utilisés : un seul poste pour le binôme (celui de Mya) : Windows (invite de commandes `cmd`), Docker Desktop (WSL 2), Docker 29.8.0, kubectl v1.36.1, kind v0.33.0.

## Palier 1

**1.1 :** Le bloc **Client** est le programme `docker` que l'on tape dans le terminal : il envoie les ordres. Le bloc **Server** est le moteur Docker (le démon) qui construit les images et fait tourner les conteneurs. Un bloc Server absent signifie que le client ne peut joindre aucun moteur, par exemple parce que Docker Desktop n'est pas démarré. C'est ce qui nous est arrivé (`failed to connect to the docker API`) ; après avoir lancé Docker Desktop, `docker run --rm hello-world` a affiché `Hello from Docker!`.

**1.2 :** Dans `-p 8080:80`, le premier nombre (8080) est le port de **notre machine** et le second (80) est le port **du conteneur**. `docker ps` affichait `0.0.0.0:8080->80/tcp`.

**1.3 :** Une image est un modèle figé et réutilisable, qui reste sur la machine. Un conteneur est une instance en cours d'exécution créée à partir d'une image, avec son propre identifiant (le `hostname` du conteneur était `cc61bbc72043`, identique à l'ID de `docker ps`). Après `docker stop`, le conteneur existait encore (`Exited`) ; après `docker rm`, il avait disparu, mais l'image `nginx:alpine` était toujours présente dans `docker image ls`.

## Palier 2

**2.2 :** Non, `EXPOSE 8080` ne suffit pas. Cette instruction sert seulement à **documenter** le port d'écoute de l'image. Pour accéder à la page depuis le navigateur, il faut **publier** le port avec `-p` (ou `ports:` dans Compose).

**2.3 :** Le `.` à la fin de `docker build` indique le **contexte de build** : le dossier courant, où Docker cherche le `Dockerfile` et les fichiers à copier (`index.html`). On utilise `-p 8080:8080` parce que l'image `nginx-unprivileged` écoute sur le port **8080** dans le conteneur, alors que `nginx:alpine` du palier 1 écoutait sur le port 80.

**2.4 :** La page affichait encore la 1.0 parce que `docker build` crée une **nouvelle image** sans modifier le conteneur déjà lancé, qui continuait de tourner avec l'image 1.0. Pour passer à la 2.0, nous avons supprimé le conteneur puis en avons lancé un nouveau à partir de la 2.0 :

```
docker rm -f petitpanier
docker run --name petitpanier -d -p 8080:8080 petitpanier:2.0
```

`docker image ls petitpanier` liste ensuite les tags `1.0` et `2.0`.

## Palier 3

**3.2 :** `Up` signifie seulement que le processus du conteneur tourne. `Up (healthy)` signifie qu'en plus le **test de santé a réussi**, donc que le site répond vraiment. Cette différence vient du bloc `healthcheck` de `compose.yaml` (un `wget` toutes les 10 secondes). Observation réelle : avec `http://localhost:8080/`, le test échouait (`Connection refused`, état `unhealthy`, vu avec `docker inspect --format "{{json .State.Health}}" petitpanier-app-1`). En remplaçant `localhost` par `127.0.0.1` dans la ligne `test`, le conteneur est passé à `Up (healthy)`. Le `compose.yaml` rendu contient donc `127.0.0.1`.

**3.3 :** La seconde commande échoue parce que, dans le conteneur `busybox`, `localhost` désigne **busybox lui-même**, où rien n'écoute sur le port 8080, et non le site. `app` est le **nom du service** défini dans `compose.yaml` : Compose le transforme en nom DNS dans le réseau `petitpanier_default`, et il pointe vers le conteneur du site (la première commande a affiché le HTML de la page).

## Palier 4

**4.1 :** Le cluster kind tourne **dans un conteneur**, il ne voit donc pas les images stockées sur notre machine : `kind load docker-image` les copie à l'intérieur du cluster. En entreprise, on pousse les images dans un **registre** (Docker Hub, GitLab Registry, etc.) et le cluster les télécharge depuis là.

**4.2 :** Les labels sont des étiquettes qui relient les objets : le Deployment les place sur ses Pods, et le Service s'en sert (`selector`) pour savoir quels Pods viser. Si le `selector` du Service était différent, il ne trouverait aucun Pod : la liste des endpoints serait vide et le site inaccessible, même si les Pods tournent (c'est la panne promo-3).

**4.3 :** L'adresse affichée dans ENDPOINTS (`10.244.0.5:8080`) est l'**adresse IP du Pod** et son port : c'est là que le Service redirige le trafic. On a besoin de `port-forward` parce que le Service est de type `ClusterIP` : il n'est joignable que depuis l'intérieur du cluster ; `port-forward` crée un tunnel entre notre machine et le Service.

## Palier 5

**5.1 :** Non, les 3 Pods ne sont pas sur des nœuds différents : la colonne NODE affiche `petitpanier-control-plane` pour les trois, car le cluster kind n'a qu'**un seul nœud**. On ne peut donc pas parler de haute disponibilité : si un Pod tombe, les autres répondent, mais si le nœud tombe, tout tombe. Il faudrait plusieurs nœuds.

**5.2 :** C'est le **Deployment** (via son ReplicaSet) qui a recréé le Pod. Le fichier demande `replicas: 3`, c'est l'**état désiré**. En supprimant un Pod, l'état réel est tombé à 2 ; Kubernetes compare en permanence l'état réel à l'état désiré et a créé un nouveau Pod (`vk6hc`) pour revenir à 3. Dans `kubectl get pods -w`, on a vu `Terminating`, puis le nouveau Pod en `Pending`, `ContainerCreating` et `Running`.

**5.3 :** `web` est le **nom du conteneur** dans le Pod (la ligne `- name: web` du `deployment.yaml`) : la commande signifie « dans le conteneur `web`, utilise l'image `petitpanier:2.0` ». Avant le `rollout undo`, `kubectl rollout history` affichait **2 révisions** (1 et 2). Après le `undo`, il affiche les révisions **2 et 3** : le `undo` n'a pas supprimé la révision 2, il a repris l'ancienne révision 1 (image 1.0) comme nouvelle révision 3.

## Palier 6 — tableau de diagnostic

| Panne | Symptôme | Hypothèse | Commande de preuve et résultat | Correction |
|---|---|---|---|---|
| promo-1 | Pod `Pending`, état `ImagePullBackOff`, conteneur jamais démarré | Le nom ou le tag de l'image est incorrect | `kubectl describe pod promo-1-dffbdc754-9mtlc` : dans Events, `Failed to pull image "petitpanier:1.O"` et `repository does not exist` | Dans `panne-1.yaml`, `1.O` (lettre O) remplacé par `1.0` (chiffre zéro), puis `kubectl apply` : Pod `Running` et `1/1` |
| promo-2 | Le Pod redémarre en boucle (`CrashLoopBackOff`), RESTARTS qui augmente (4 observés) | Le conteneur démarre puis s'arrête avec une erreur : la commande lancée finit par `exit 1` | `kubectl describe pod promo-2-7f48f789dd-pwnf9` : `Exit Code: 1`, `Restart Count: 4` ; `kubectl logs promo-2-7f48f789dd-pwnf9` : `ERREUR : variable PROMO_CODE absente` | Ligne `command` supprimée de `panne-2.yaml` (l'image lance alors nginx, qui reste en marche), puis `kubectl apply` : Pod `Running` et `1/1` |
| promo-3 | Pods `Running` et `1/1`, mais site inaccessible et `Endpoints` vide | Le `selector` du Service ne correspond pas aux labels des Pods | `kubectl describe service promo-3` : `Selector: app=promo-three`, `Endpoints:` vide ; `kubectl get pods --show-labels` : `app=promo-3` | `selector: { app: promo-three }` remplacé par `{ app: promo-3 }`, puis `kubectl apply` : `kubectl get endpoints promo-3` affiche 2 adresses (`10.244.0.20:8080` et `10.244.0.21:8080`) |

**6.2 (--previous) :** Observation réelle : `kubectl logs promo-2-... --previous` a répondu `unable to retrieve container logs` (le Pod avait déjà redémarré plusieurs fois), alors que `kubectl logs promo-2-...` seul a affiché le message d'erreur. Principe : `kubectl logs` lit le conteneur **en cours** ; pendant l'attente entre deux redémarrages (`CrashLoopBackOff`), il peut être vide, et `--previous` sert à lire les logs du conteneur précédent, quand ils sont encore disponibles.

**6.3 (pourquoi Running mais inaccessible) :** Les Pods tournaient correctement, mais un Service ne trouve ses Pods que par leurs **labels**. Le `selector` visait `promo-three` alors que les Pods portaient `promo-3` : le Service n'avait aucun endpoint, donc aucun trafic ne leur arrivait.

## Palier 7

| Étape | Commande | Si échec |
|---|---|---|
| build | `docker build -t petitpanier:2.0 .` | L'image n'est pas créée et le pipeline s'arrête : rien n'est livré, la version précédente continue de tourner. |
| test | `docker compose up -d` puis `docker compose ps` (attendre `Up (healthy)`) | Le site ne répond pas correctement : le pipeline s'arrête avant la publication, aucune version défectueuse n'arrive au cluster. |
| publish | `kind load docker-image petitpanier:2.0 --name petitpanier` | Le cluster ne trouve pas l'image : les Pods restent en `ImagePullBackOff` (comme promo-1) et le déploiement échoue. |
| deploy | `kubectl apply -f k8s/` | Le déploiement échoue (`kubectl rollout status` signale l'erreur) et les anciens Pods restent actifs ; on peut revenir en arrière avec `kubectl rollout undo`. |

Remarque : dans une vraie entreprise, `publish` serait un `docker push` vers un registre ; `kind load` est l'équivalent local.

**7.1 :** Un pipeline tague l'image avec l'identifiant du commit pour trois raisons. (1) Chaque version a un **tag unique** : avec `latest`, deux versions différentes portent le même nom. (2) Kubernetes **détecte le changement** : au 5.3, `kubectl set image ... web=petitpanier:2.0` a modifié le nom de l'image dans le Deployment, ce qui a déclenché la mise à jour ; avec un tag toujours identique, rien ne change dans le Deployment, et avec `imagePullPolicy: IfNotPresent` un nœud qui possède déjà `latest` ne retélécharge pas la nouvelle image. (3) Le **retour arrière est précis** : avec `rollout undo`, on sait exactement quelle version on retrouve et on peut la relier au commit dans Git.

**Workflow n8n :** Webhook GitLab (déclencheur : reçoit l'événement « pipeline terminé », avec un secret partagé pour vérifier l'expéditeur) → IF `statut = failed` (condition : si le statut est `success`, le workflow s'arrête) → message Slack dans `#petitpanier-ops` (action : toute l'équipe est prévenue, avec le projet, la branche, l'étape en erreur et le lien vers le pipeline) → e-mail à la personne d'astreinte (action facultative). Les identifiants (token Slack, secret du webhook, mots de passe) ne sont jamais écrits dans les fichiers ni dans le workflow : ils sont rangés dans le gestionnaire d'identifiants (*Credentials*) chiffré de n8n, et côté GitLab dans des variables CI/CD masquées.

## Défis réalisés

- **Défi 1 :** deux conteneurs nginx sur les ports 8081 et 8082. Impossible de mettre les deux sur 8081 : le port de la machine ne peut être occupé que par un seul programme à la fois (Docker répond `port is already allocated`).
- **Défi 2 :** `docker build -t petitpanier:2.0 .` relancé sans modification : les étapes `FROM` et `COPY` sont `CACHED` et le build est quasi instantané, car Docker réutilise le résultat d'une instruction quand rien n'a changé.
- **Défi 3 :** service `stats` ajouté à `compose.yaml` ; `docker compose exec stats wget -qO- http://app:8080` affiche la page : les deux services sont dans le même réseau et `app` est résolu par DNS.
- **Défi 4 :** dans la section Events de `kubectl describe pod`, `Scheduled` (le planificateur choisit un nœud), `Pulled` ou « already present on machine » (l'image est récupérée ou déjà présente), `Created` (le conteneur est créé), `Started` (il démarre).
- **Défi 5 :** on préfère modifier le fichier parce qu'il est versionné (Git), relu par l'équipe et rejouable ; `kubectl scale` change le cluster sans laisser de trace dans le fichier.
- **Défi 6 :** avec `path: /sante`, les nouveaux Pods restent à `0/1` bien que `Running` ; `describe` indique `Readiness probe failed` (code 404) car la page `/sante` n'existe pas.
- **Défi 7 :** esquisse de `.gitlab-ci.yml` :

```yaml
stages: [build, test, publish, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE .

test:
  stage: test
  script:
    - docker compose up -d
    - docker compose ps

publish:
  stage: publish
  script:
    - docker push $IMAGE

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/petitpanier web=$IMAGE
    - kubectl rollout status deployment/petitpanier
```
