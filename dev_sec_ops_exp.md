# Mon recueil de batailles techniques

*Un journal des bugs, des impasses et des victoires qui ont façonné mon parcours de dev.*

---

## Pourquoi ce recueil

Chaque problème technique qu'on résout laisse une trace : une intuition affûtée, un réflexe de diagnostic, une leçon qu'on ne veut plus jamais réapprendre à la dure. Ce recueil sert à garder ces traces avant qu'elles ne s'effacent. Ce n'est pas une documentation technique froide — c'est l'histoire de ce que j'ai vécu en construisant, cassant, et réparant mes systèmes.

Chaque entrée raconte : le contexte dans lequel le problème est apparu, ce que j'ai ressenti en le découvrant, comment j'ai creusé, ce qui a fini par marcher, et ce que j'en retiens.

---

## Entrée #1 — Du 22 au 23 août 2026 — Le mystère de la collection Qdrant qui n'existait jamais

**Contexte :** Je venais de déployer mon application backend RAG sur Dokploy, avec MinIO pour le stockage des documents et Qdrant comme base vectorielle — les deux via les services proposés directement par Dokploy. Tout semblait en ordre : les conteneurs tournaient, l'interface web de gestion du RAG était accessible, l'authentification fonctionnait. J'étais dans cette phase satisfaisante où l'infrastructure semble enfin stable après le sprint de déploiement.

**Le problème :** J'ai uploadé un premier fichier via mon appli web. Message de succès : *« bien indexé »*. Sauf qu'en allant vérifier dans le dashboard Qdrant, rien. Pas de collection, pas d'index, pas le moindre point vectoriel. Le vide total, alors que l'application m'assurait que tout allait bien. Ce genre de décalage entre ce que l'app affirme et ce qui existe réellement est particulièrement déstabilisant — on ne sait même pas par quel bout commencer à douter.

**L'enquête :** J'ai commencé par fournir les logs et la configuration. Les logs révélaient une ligne qui changeait toute la lecture du problème : `[Errno 110] Connection timed out`, répétée à chaque tentative de contact avec Qdrant, sur plusieurs minutes d'affilée. Pas une erreur d'authentification, pas un refus de connexion — un vrai silence réseau. Premier réflexe : tester la connectivité brute avec `curl` (absent du conteneur), puis avec un one-liner Python (`httpx`). Résultat troublant : **ça a marché du premier coup**, `200 OK`, réponse quasi instantanée. Le service applicatif, lui, continuait à timeout en boucle, y compris lors d'un vrai upload de document qui a fini avec le statut `stored_only` — stocké dans MinIO, jamais indexé.

Ce paradoxe (un test manuel qui réussit, une app qui échoue en continu) a orienté l'enquête vers autre chose qu'un simple problème de firewall ou de DNS mort : quelque chose d'intermittent, ou une différence structurelle entre les deux chemins réseau. La bascule décisive a été de sortir du conteneur applicatif pour regarder l'hôte lui-même avec `docker ps` et `docker network ls` — remonter d'un niveau d'abstraction quand on tourne en rond dans un conteneur.

**Le déclic :** En listant les conteneurs, les noms ont tout dit : `testapp-backend-yesagw`, `testapp-qdrant-rem4ui-qdrant-1`, `testapp-minioapp-7vxvpe-minio-1` — trois suffixes aléatoires différents, trois stacks Dokploy complètement indépendants. En inspectant `dokploy-network` (le réseau overlay partagé de la plateforme), le backend y était bien présent. Qdrant, non. Il vivait isolé sur son propre réseau bridge local, `testapp-qdrant-rem4ui`, jamais rattaché au réseau commun. Le backend et Qdrant n'avaient tout simplement jamais pu se parler en interne — toute communication devait obligatoirement sortir vers internet, traverser Traefik, et revenir sur le même serveur. Un aller-retour fragile, sujet à toutes sortes d'instabilités, qui expliquait à la fois les timeouts en rafale et le succès isolé du test manuel.

**La résolution :** Édition du fichier `docker-compose` de Qdrant dans Dokploy pour y ajouter explicitement le réseau externe partagé :

```yaml
networks:
  - dokploy-network

networks:
  dokploy-network:
    external: true
```

Redéploiement du service Qdrant. Vérification que le conteneur apparaissait bien désormais dans `dokploy-network`. Puis, depuis le backend, un test de résolution DNS (`getent hosts qdrant`) a confirmé une IP interne valide, et un test de port (`/dev/tcp/qdrant/6333`) a répondu `OK` instantanément. Dernière étape : basculer la variable d'environnement `QDRANT_URL` de l'URL publique HTTPS vers l'adresse interne `http://qdrant:6333`, redéployer le backend, et réuploader un document de test. Cette fois, la collection est apparue dans Qdrant avec ses points, en quelques secondes.

**Ce que j'en retiens :**
- Un `Connection timed out` (par opposition à un `refused`) est presque toujours un problème de routage ou d'isolation réseau, pas d'authentification ou de config applicative — ça vaut le coup de s'en souvenir pour gagner du temps la prochaine fois.
- Un test manuel qui réussit alors que l'app échoue en boucle est un signal fort d'instabilité *intermittente*, pas d'un blocage total — ça oriente immédiatement vers l'infrastructure réseau plutôt que vers le code.
- Sur Dokploy (et plus largement avec Docker Swarm), chaque service déployé séparément peut vivre dans son propre réseau isolé par défaut. Il ne faut jamais supposer que deux services de la même « app logique » se voient automatiquement — toujours vérifier avec `docker network inspect`.
- Faire communiquer deux services sur le même serveur en repassant systématiquement par le domaine public HTTPS (au lieu du réseau interne) est un anti-pattern classique : plus lent, plus fragile, et ça peut créer des instabilités qui ressemblent à des bugs applicatifs alors que ce n'en est pas un.
- La bascule décisive dans le diagnostic n'est pas venue du code ou des logs applicatifs, mais du fait de sortir du conteneur pour regarder la topologie réseau depuis l'hôte. Quand on tourne en rond à l'intérieur d'une couche, il faut parfois prendre du recul et regarder la couche du dessous.

**Tags :** `#docker` `#dokploy` `#qdrant` `#réseau` `#rag` `#déploiement`

---

## Entrée #2 — 25 septembre 2026 — Le login qui ne marchait que sur le serveur

**Contexte :** J'avais déployé une application d'anatomopathologie en local à l'hôpital HOGIP (ex-CTO), sur un serveur Docker à l'adresse `172.16.0.5`. Quatre conteneurs dans un même `docker-compose` : PostgreSQL, un backend Spring Boot (port `8080`), un frontend React servi sur le port `80`, et MinIO. Le frontend avait été buildé avec `VITE_API_URL=http://172.16.0.5:8080/api`. Sur le serveur lui-même, tout fonctionnait : page de login, authentification, accès à l'interface d'administration.

**Le problème :** Depuis les autres postes du même réseau, le frontend s'affichait bien dans le navigateur, mais l'authentification échouait systématiquement. Même application, mêmes identifiants, même réseau : ça passait sur le serveur et nulle part ailleurs. J'ai fini par me déplacer de chez moi jusqu'au service informatique de l'hôpital pour enquêter sur place.

**L'enquête :** Trois pistes classiques pour ce symptôme « ça marche sur le serveur, pas sur les clients » :
1. Une URL d'API contenant encore `localhost`, figée dans le bundle au moment du build (les variables `VITE_*` sont injectées au build, pas au démarrage du conteneur). Sur le serveur, `localhost` désigne le serveur ; sur un client, il désigne le client lui-même.
2. Une configuration CORS du backend qui n'autorise que l'origine `http://localhost` et rejette `http://172.16.0.5`.
3. Un pare-feu sur le serveur qui laisse passer le port `80` mais pas le `8080`.

J'ai ouvert les DevTools (F12 → onglets *Network* et *Console*) sur un poste client pendant une tentative de connexion. Les requêtes partaient bien vers `http://172.16.0.5:8080/api`, et non vers `localhost`. Première piste écartée : le build du frontend était correct.

**Le déclic :** Le front s'affichait (port `80` joignable) mais les appels API échouaient (port `8080`). En consultant les règles entrantes du pare-feu du serveur, le constat était sans appel : **seul le port 80 était ouvert**. Sur le serveur, les appels vers `8080` restaient locaux et ne traversaient jamais le pare-feu ; depuis les autres machines, ils étaient bloqués.

**La résolution :** Ajout d'une règle entrante autorisant le port TCP `8080` sur le pare-feu du serveur. Test immédiat depuis une autre machine du réseau : l'authentification passe et l'interface d'administration s'ouvre. Aucune modification du code, du `docker-compose` ou des images n'a été nécessaire.

**Ce que j'en retiens :**
- « Ça marche sur le serveur mais pas depuis les clients » : le trafic du serveur vers lui-même ne passe pas par les mêmes règles que le trafic venant du réseau. Tester depuis le serveur ne prouve rien pour les clients — il faut toujours tester depuis une autre machine.
- Un frontend qui s'affiche ne prouve que l'ouverture de *son* port. Avec un front et une API sur des ports différents, il faut vérifier chaque port séparément (`curl http://172.16.0.5:8080/...` depuis un client aurait suffi).
- Les DevTools sont le premier réflexe : l'onglet *Network* montre l'URL réellement appelée et le type d'échec (timeout, CORS, 401…), ce qui élimine des pistes en quelques secondes.
- Publier un port avec Docker (`ports: "8080:8080"`) ne l'ouvre pas pour autant dans un pare-feu en amont : ce sont deux couches distinctes à vérifier.
- Piste d'amélioration : faire passer l'API derrière un reverse proxy Nginx dans le conteneur frontend (`VITE_API_URL=/api`, `proxy_pass http://backend:8080`). Un seul port exposé, plus de CORS, plus d'IP en dur — et une règle de pare-feu en moins à oublier.
- Documenter les règles de pare-feu nécessaires au déploiement, et obtenir un accès distant (VPN, SSH) au serveur : un problème de port ne devrait pas coûter un déplacement.

**Tags :** `#docker` `#pare-feu` `#réseau` `#spring-boot` `#react` `#vite` `#déploiement` `#on-premise`

---

## Entrée #3 — Du 28 au 29 septembre 2026 — Le TACOJO et le code « 007 » qui ne voulait rien dire ailleurs

**Contexte :** Je pilotais Hopitalia, la refonte en Spring Boot / PostgreSQL d'une ancienne application hospitalière écrite en WinDev. Le même backend était déployé dans plusieurs hôpitaux (Albert Royer, Dantec…) et centres de santé, chacun avec sa propre base de données. Parmi les écrans à porter : le TACOJO, un rapport du laboratoire qui comptait les actes réalisés sur une période, séparément pour les patients externes et pour les patients hospitalisés. Un développeur backend l'avait déjà porté, mais le résultat ne m'inspirait pas confiance.

**Le problème :** Dans WinDev, les deux requêtes du TACOJO contenaient la même ligne : `PRESTATION.CODE_SPECIALITE = '007'`. Un code figé, valable dans une seule base. Le portage Java avait supprimé le `007`… pour le remplacer par d'autres fragilités :
- une méthode `resolveCategorieAndCode` qui « devinait » la catégorie à partir de la spécialité en prenant `rows.get(0)` d'une liste sans ordre garanti — la catégorie choisie par l'utilisateur pouvait être silencieusement remplacée par une autre ;
- des filtres où « vide » voulait dire « pas de filtre » : un appel sans paramètre comptait tous les actes de l'hôpital, sans la moindre erreur ;
- deux nouveaux codes en dur, `NOT IN ('D23', '55524')` ;
- des jointures par nom (`nom_specialite = :nomServiceSalle`) au lieu des identifiants, et des prestations d'hospitalisation sans salle qui disparaissaient du rapport sans laisser de trace.

**L'enquête :** Plutôt que de corriger le code Java ligne par ligne, je suis reparti des trois fichiers WinDev d'origine (`REQUETE_Tacojo`, `REQ_TACOJO_HOSP`, `BTN OK`) pour comprendre ce qu'ils faisaient vraiment. Quelques pièges sont apparus : l'alias `REF_ACTE` désignait en fait un `COUNT` ; le combo nommé `COMBO_specialite` filtrait sur `ACTE.CATEGORIE` ; la boucle hospitalisation relisait salle et spécialité avec `HLitRecherche` et, en cas d'échec, réutilisait le buffer de la ligne précédente — un acte pouvait être compté dans le mauvais service.

Ensuite, j'ai reformulé la question métier sans le `007`. Au labo, la spécialité n'était pas un critère : *c'était* le labo. La question devenait : **combien de fois chaque acte d'une section du labo a-t-il été facturé sur la période, selon la provenance du patient ?** Même le vocabulaire comptait : on comptait des *actes facturés* (des prestations), pas des *analyses réalisées* — ce n'était pas la même chose si un acte regroupait plusieurs analyses ou si une prestation était annulée.

**Le déclic :** Tout a basculé quand j'ai interrogé la vraie base au lieu de raisonner sur le code. Dans Hopitalia, le laboratoire était un *service* (`LABORATOIRE`) qui regroupait trois spécialités : Biochimie, Hématologie, Bactériologie. Le `007` de WinDev, c'était la Bactériologie. Les « sections » du labo étaient donc des spécialités, et non des catégories d'actes (qui contenaient des valeurs génériques comme `EXAMEN` ou `CONSULTATION`).

Première correction : j'ai déplacé le code du service labo (`LABO01`) dans `application.yml`, surchargeable par variable d'environnement. Ça me semblait propre… jusqu'à ce que je réalise que ce n'était qu'un `007` déplacé. D'un hôpital à l'autre, les codes des services, des spécialités et des actes étaient complètement différents. Une valeur par défaut dans un fichier de déploiement restait une donnée métier figée.

**La résolution :** Le périmètre du rapport est passé dans les données, paramétrables depuis l'application par chaque hôpital :
- une case `service.laboratoire` : les spécialités des services cochés devenaient les sections du TACOJO, quel que soit leur code ;
- une case `acte.exclu_tacojo` pour les actes à ne jamais compter (en remplacement de `D23` / `55524`) ;
- si aucun service n'était coché, l'API répondait désormais `409` avec un message explicite, au lieu d'un rapport vide trompeur.

Les deux endpoints ont été remplacés par un seul `GET /api/v1/laboratoire/tacojo`, qui renvoyait directement la matrice acte × provenance (Externes, puis une colonne par service d'hospitalisation). Côté SQL, une seule requête rangeait chaque prestation dans **une et une seule** provenance, exclusions comprises :

```sql
CASE
  WHEN e.type = :typeHospitalisation THEN
    CASE WHEN sp_hosp.id IS NULL THEN 'HOSPITALISATION_SANS_SERVICE'
         ELSE 'HOSPITALISATION' END
  WHEN e.type IN (:typesExternes) THEN
    CASE WHEN COALESCE(e.passer_par_sau, FALSE)    THEN 'EXCLU_SAU'
         WHEN COALESCE(e.paiement_modalite, FALSE) THEN 'EXCLU_MODALITE'
         WHEN NOT COALESCE(p.paye, FALSE)          THEN 'EXCLU_NON_PAYE'
         ELSE 'EXTERNE' END
  ELSE 'EXCLU_TYPE_ENTREE_INCONNU'
END AS provenance
```

Cela a donné un invariant vérifiable : `totalGeneral + exclus.total` = nombre de prestations du labo sur la période. Enfin, un `TacojoRunner` recalculait chaque règle WinDev par une requête SQL indépendante et la comparait au service : 30 vérifications au vert sur la base de test. Une note a été rédigée pour le développeur frontend.

**Ce que j'en retiens :**
- Supprimer un code en dur ne suffit pas : il faut se demander *à qui* appartient l'information. `007` → `LABO01` dans le yml, c'était le même problème déplacé d'une couche.
- En déploiement multi-sites, toute donnée qui varie d'une base à l'autre doit vivre dans la base et être modifiable depuis l'application — ni dans le code, ni dans la configuration de déploiement.
- Avant de porter du code legacy, reformuler la question métier en une phrase. Elle révèle les critères inutiles (la spécialité) et les confusions de vocabulaire (acte / analyse, spécialité / catégorie).
- Interroger les vraies données tôt : une requête sur la base a tranché en quelques secondes ce que des heures de lecture de code n'auraient pas établi.
- Un paramètre deviné (`rows.get(0)`) est pire qu'une erreur. Un rapport ne doit jamais rien perdre en silence : refuser explicitement, et rendre visible ce qui est exclu.
- Les règles d'un système legacy se protègent mieux par des assertions indépendantes que par la relecture : si le rapport et le recalcul divergent, le Runner le dit.

**Tags :** `#spring-boot` `#postgresql` `#windev` `#migration` `#legacy` `#multi-sites` `#sql` `#laboratoire`

---

## Entrée #4 — *(à venir)*

*...*
