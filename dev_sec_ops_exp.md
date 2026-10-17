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

## Entrée #4 — 29 septembre 2026 — Les résultats de laboratoire qui montraient le mauvais patient

**Contexte :** Dans Hopitalia, la page « Résultats Labo Plus » affichait tous les résultats de laboratoire d'un patient, issus de cinq sources : l'automate Hoggy, les examens manuels de bactériologie (stockés en JSON), la sérologie VIH, la toxoplasmose/rubéole et l'HGPO. Côté backend, tout reposait sur une seule requête SQL native : une `UNION ALL` de cinq branches, 842 lignes au total, accompagnée de sa propre requête de comptage pour la pagination. Côté front, l'index du patient était calculé à partir de quinze champs possibles, avec « nom prénom » en dernier recours.

**Le problème :** Je n'avais que des bugs sur cette page, et chaque correction semblait en faire apparaître un autre. L'onglet Historique ne chargeait pas, la pagination affichait des totaux incohérents, des lignes apparaissaient en double. J'avais l'intuition que la complexité des requêtes était en cause, sans savoir par où commencer.

**L'enquête :** Plutôt que de relire la requête, j'ai reproduit chaque symptôme par un vrai appel sur la base de test. Chaque bug a trouvé sa cause :
- L'onglet Historique envoyait `sort=idResultatLigne`. Spring Data, face à une requête native, préfixait ce champ inconnu par l'alias qu'il croyait détecter : `e.idResultatLigne`. PostgreSQL répondait `missing FROM-clause entry for table "e"`.
- Pour un patient à 136 lignes, `totalElements` valait successivement 50, 100, 136 puis 12 selon la page. La requête de comptage n'éclatait pas le JSON de la branche bactério (`jsonb_each`), contrairement à la requête principale : les deux copies avaient divergé.
- Le pire : deux fiches homonymes (même nom, même prénom, index différents) voyaient les résultats l'une de l'autre, et chaque ligne apparaissait en double. En cause, une jointure de rapprochement présente dans chaque branche :

```sql
LEFT JOIN fiche_patient fp
  ON (fp.nom = e.nom AND fp.prenom = e.prenom)
  OR (fp.nin IS NOT NULL AND fp.nin IN (e.nin, e.numcin, e.matricule, ...))
  OR (fp.matricule IS NOT NULL AND fp.matricule IN (...))
  OR ...
```

D'autres bombes attendaient leur heure. La source d'une ligne (automate, VIH…) était déduite de plages d'identifiants négatifs (`-100000`, `-200000`…) : au-delà de 100 000 détails bactério, ils seraient devenus des « VIH ». Des sérologies VIH non validées s'affichaient sans aucune mention. Un résultat sans date recevait `CURRENT_DATE` et apparaissait comme « du jour ». Enfin, le bloc d'identification du patient était copié vingt fois (5 branches × 4 requêtes) : la divergence du comptage n'était que la première.

**Le déclic :** En interrogeant la base, j'ai constaté que `fiche_patient.index` était **unique et obligatoire**, et que 553 entrées sur 562 portaient ce même index. Le lien entre un patient et ses passages existait déjà, simple et sûr : `entree."index" = fiche_patient."index"`. Toutes ces jointures par nom, NIN ou matricule, censées « rattraper » des cas, étaient non seulement inutiles, mais dangereuses. La complexité n'était pas un décor autour des bugs : c'était le bug.

**La résolution :** La requête géante a été remplacée par une architecture en trois temps :
- **Identifier le patient une seule fois** : l'index exact ; à défaut, un NIN ou un matricule accepté seulement s'il ne désignait qu'un seul patient (sinon `409`) ; un identifiant inconnu donnait `404` au lieu d'une page vide. Plus aucun rapprochement par nom.
- **Une requête courte par source**, filtrée par `WHERE e."index" = :index`, chacune déclarant elle-même sa source. Chaque ligne recevait une clé unique (`BACTERIO_MANUEL:205:glycemie`).
- **Fusion, tri et pagination en Java** : un patient ayant au plus quelques centaines de lignes, la pagination en mémoire était exacte, sans requête de comptage. Le tri n'acceptait qu'une liste blanche de champs, avec une erreur `400` explicite pour les autres.

Les résultats non validés ont été conservés mais marqués d'un statut de validation, une date absente est restée `null`, et l'onglet Jour a reçu un vrai paramètre `date`. Les URLs et la structure de réponse n'ont pas changé. Un Runner a vérifié 120 assertions sur la base de test, dont l'absence de fuite entre les deux homonymes, et une note a été rédigée pour le développeur frontend.

Dernier rebondissement : le développeur frontend a signalé une erreur `500` (`NoClassDefFoundError` sur l'ancien mapper) et l'ancienne erreur SQL. Le serveur lancé depuis l'IDE tournait depuis 01:11, avant la refonte, avec l'ancien code en mémoire ; entre-temps, les anciens `.class` avaient été supprimés de `target/classes`. Un simple redémarrage a suffi.

**Ce que j'en retiens :**
- Face à une avalanche de bugs, reproduire chaque symptôme par un appel réel avant de toucher au code : la liste des causes vient des faits, pas de la relecture.
- Un bloc copié vingt fois finit toujours par diverger. Une requête de comptage écrite à la main à côté d'une `UNION`, c'est deux vérités à maintenir.
- Ne jamais identifier un patient par son nom et son prénom. Dans une application médicale, afficher les résultats d'un autre patient est le pire bug possible : il faut utiliser l'identifiant unique qui existe déjà, et refuser (`404`, `409`) plutôt que deviner.
- Ne jamais encoder une information dans des plages d'identifiants : la source d'une ligne doit être une donnée explicite.
- Quand le volume par clé est borné, une pagination en mémoire exacte vaut mieux qu'une pagination SQL « astucieuse » qui ment sur ses totaux.
- Ne rien masquer ni inventer : un résultat non validé s'affiche marqué, une date absente n'est pas la date du jour.
- Avant de conclure qu'une correction ne marche pas, vérifier quel build tourne réellement (processus, heure de démarrage). Et ne jamais toucher à `target/classes` sous un serveur actif.

**Tags :** `#spring-boot` `#spring-data-jpa` `#postgresql` `#sql` `#pagination` `#securite-patient` `#legacy` `#refactoring`

---

## Entrée #5 — 2 octobre 2026 — La carte PASTEF que la prod cherchait sur ma machine

**Contexte :** MONCAP, la plateforme d'adhésion en React + Vite, était déployée en ligne sur `https://moncap.innovamind.tech`, avec un backend FastAPI hébergé à part. La première étape du formulaire d'adhésion est la vérification du N° de carte PASTEF : le front appelle `/api/v1/adhesions/verifier-carte-pastef`, et les autres champs ne se débloquent qu'une fois la carte validée. Je venais de livrer une mise à jour du formulaire : `npm run build`, puis envoi du dossier `dist` sur l'hébergement.

**Le problème :** Sur la version en ligne, la recherche de carte ne fonctionnait plus. Sous le champ, le message `Failed to fetch — réessayez ou contactez un administrateur MONCAP.`, et juste en dessous un encadré rouge : *« Carte PASTEF invalide ou introuvable »*. Pour un visiteur, le verdict était clair : sa carte n'existait pas. Sauf que la carte était bonne — c'était la requête qui n'aboutissait jamais.

**L'enquête :** `Failed to fetch` n'est pas une réponse du serveur : c'est le navigateur qui dit qu'il n'a reçu aucune réponse. Inutile donc de soupçonner la carte ou la logique métier. J'ai ouvert les DevTools sur la page en ligne, onglet *Network*, et sélectionné la requête `verifier-carte-pastef`. Dans *Headers*, la ligne qui a tout réglé :

```
Request URL : http://localhost:8000/api/v1/adhesions/verifier-carte-pastef
Origin      : https://moncap.innovamind.tech
```

Accompagnée de l'avertissement *« Provisional headers are shown »* : la requête n'était jamais allée au bout. Le site public, servi en HTTPS, appelait une API sur `localhost`. Chez un visiteur, `localhost` désigne sa propre machine, où rien n'écoute sur le port `8000`.

**Le déclic :** Direction le fichier `.env` :

```
VITE_API_URL="http://localhost:8000"

# VITE_API_URL="https://thior.alwaysdata.net"
```

J'avais basculé sur `localhost` pour travailler en local, et lancé `npm run build` sans revenir en arrière. Or les variables `VITE_*` ne sont pas lues au démarrage : Vite les remplace par leur valeur **au moment du build**, en texte brut dans le JavaScript. Le `dist` que j'avais déployé contenait mon adresse de développement, gravée dans le bundle. Rien côté hébergement ne pouvait la corriger après coup.

**La résolution :** Inverser les deux lignes du `.env` (adresse de prod active, `localhost` en commentaire), relancer `npm run build`, redéployer le `dist`. Vérification sans passer par le formulaire : le bundle servi en ligne (`main-BfWYKIGr.js`) contenait bien `https://thior.alwaysdata.net` comme base de l'API, et la recherche de carte refonctionnait.

**Ce que j'en retiens :**
- Avec Vite, `.env` n'est pas une configuration d'exécution : c'est une entrée du build. Ce qui est dans le fichier au moment de `npm run build` part en production, définitivement.
- J'avais pourtant listé ce piège dans l'entrée #2 (« une URL d'API contenant encore `localhost`, figée dans le bundle ») — sans m'en protéger. Une leçon qui reste dans un carnet ne protège de rien : elle doit devenir un garde-fou.
- Un seul fichier qu'on commente/décommente selon l'environnement, c'est une erreur programmée. Vite charge tout seul `.env.development` pour `npm run dev` et `.env.production` pour `npm run build` : chaque adresse a sa place, et plus rien à basculer à la main.
- Vérifier ce qui est réellement déployé, pas ce qu'on croit avoir buildé : un `grep localhost dist/assets` avant l'envoi, ou l'onglet *Network* juste après.
- `Failed to fetch` signifie « pas de réponse », jamais « réponse négative ». L'interface l'a pourtant affiché comme une carte invalide : une erreur réseau ne doit pas se déguiser en verdict métier, surtout devant un futur adhérent.

**Tags :** `#vite` `#react` `#env` `#build` `#deploiement` `#devtools` `#moncap`

---

## Entrée #6 — 15 octobre 2026 — Les 211 Go que Docker gardait pour lui

**Contexte :** La semaine précédente, j'avais énormément utilisé Docker : builds à répétition, images téléchargées, conteneurs lancés et relancés, sans jamais jeter un œil aux volumes ni au cache de build. Ce jour-là, je travaillais sur HermiNas et je lançais simplement `task bundle`, qui fait un `docker build -f deploy/Dockerfile --target dist -o dist .` pour produire le livrable.

**Le problème :** Le build est mort avant même de commencer :

```
[+] Building 0.0s (0/0) docker:desktop-linux
ERROR: failed to build: failed to solve: write /var/lib/desktop-containerd/daemon/io.containerd.metadata.v1.bolt/meta.db: input/output error
task: Failed to run task "bundle": exit status 1
```

Un `input/output error` sur la base de métadonnées de containerd : j'ai eu très peur, ce genre de message fait penser à un disque qui lâche. Je suis allé voir *Ce PC* : le disque local **C:** était tout rouge, **10 Go libres**. Le disque de données NETRA (**N:**), lui, avait encore 202 Go libres, alors que la veille les deux étaient à peu près au même niveau. En deux jours, C: s'était rempli. Et je me suis souvenu, trop tard, que mon collègue Mohamed Diallo m'avait parlé d'un problème de ce genre.

**L'enquête :** Premier réflexe, avant de comprendre : ouvrir Docker Desktop et supprimer à la main conteneurs, images et volumes. Puis j'ai demandé une analyse du disque C:. Le scan complet était trop lent ; la piste qui a payé, c'est de trier `AppData\Local` par date de modification. En tête : `Docker`, `wsl`, `docker-secrets-engine`, tous modifiés dans la nuit. Une recherche des disques virtuels a donné la réponse :

```
211.02 Go   AppData\Local\Docker\wsl\disk\docker_data.vhdx
 17.92 Go   AppData\Local\wsl\{74d3...}\ext4.vhdx
```

Le plus troublant : après mes suppressions dans Docker Desktop, le fichier faisait **toujours 211 Go**. J'avais tout vidé, et Windows ne voyait aucune différence.

**Le déclic :** Docker Desktop ne stocke pas ses données directement dans Windows : tout vit dans un disque virtuel `docker_data.vhdx`, monté dans la VM WSL 2. Ce fichier grossit à chaque image, chaque couche de build, chaque volume, mais **il ne rétrécit jamais tout seul**. Supprimer des images libère de la place *à l'intérieur* du disque virtuel, pas sur C:. Et le `input/output error` n'était pas un disque mourant : c'était containerd qui n'arrivait plus à écrire dans un disque virtuel qui ne pouvait plus grandir, faute de place sur l'hôte.

**La résolution :** Il fallait compacter le `.vhdx`, et chaque étape a eu son obstacle :

1. `docker builder prune -a` → `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`. Le moteur Docker était arrêté ; comme j'avais déjà tout supprimé, je suis passé directement au compactage.
2. `compact vdisk` dans `diskpart` → *« Le processus ne peut pas accéder au fichier car ce fichier est utilisé par un autre processus »*. Docker et WSL tenaient encore le fichier.
3. `wsl --shutdown` puis `Stop-Process` sur `vmmem` et `wslservice` → *Accès refusé* : mon PowerShell n'était pas lancé en administrateur.
4. Dans un PowerShell **administrateur** :

```powershell
wsl --shutdown
Stop-Service WSLService -Force
diskpart
```

```
select vdisk file="C:\Users\DELL\AppData\Local\Docker\wsl\disk\docker_data.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

`vmmem` apparaissait encore dans la liste des processus, mais avec le service WSL arrêté, le compactage est passé : **211 Go → 4,1 Go**, et C: est remonté à **308 Go libres**. Un dernier faux pas : j'avais quitté `diskpart` sans `detach vdisk`, et en tapant `select vdisk ...` directement dans PowerShell, j'ai eu une erreur `Select-Object` : `select` y est l'alias d'un cmdlet PowerShell. Les commandes `diskpart` ne marchent qu'à l'invite `DISKPART>`.

**Ce que j'en retiens :**
- Supprimer dans Docker Desktop ne rend pas l'espace à Windows. Le vrai coupable se mesure dans `AppData\Local\Docker\wsl\disk\docker_data.vhdx`, et se règle par un compactage.
- Mesurer avant de supprimer : `docker system df` montre ce qui pèse vraiment (images, conteneurs, volumes, **cache de build**). J'ai tout effacé à l'aveugle, volumes compris, alors que le cache de build était sans doute le plus gros morceau.
- Un `input/output error` dans Docker n'annonce pas forcément un disque qui meurt : vérifier d'abord l'espace libre sur l'hôte.
- Prévenir plutôt que guérir : plafonner la taille du disque dans *Settings → Resources → Advanced* (ou le déplacer sur N:), activer le ramasse-miettes du builder (`"builder": { "gc": { "enabled": true, "defaultKeepStorage": "20GB" } }`) et lancer de temps en temps `docker builder prune`.
- Quand un fichier est « utilisé par un autre processus », la réponse n'est pas de forcer : arrêter proprement ce qui le tient (`wsl --shutdown`, service WSL), en administrateur.
- Mohamed m'avait prévenu. Un retour d'expérience entendu, mais pas noté, ne protège de rien : c'est exactement pour ça que ce recueil existe.

**Tags :** `#docker` `#wsl2` `#windows` `#disque` `#vhdx` `#diskpart` `#build-cache` `#herminas`

---

## Entrée #7 — 17 octobre 2026 — Le port 80 que Caddy avait pris à Traefik

**Contexte :** Avec Mohamed, nous devions rencontrer un vendeur d'intrants agricoles du marché de Thiaroye pour lui présenter Agroboom. Avant de partir, chacun de chez soi, nous avons fait un dernier tour de l'application. Toute ma plateforme est déployée avec Dokploy sur un VPS Hostinger : applications web, API, bases de données, MinIO, le tout en conteneurs Docker derrière Traefik, le reverse proxy fourni par Dokploy.

**Le problème :** L'application mobile et l'application web ne fonctionnaient plus : plus de données reçues, plus d'accès. L'API elle-même ne répondait pas. À quelques heures d'une démonstration, c'est le pire moment pour s'en apercevoir. En ouvrant la console MinIO dans le navigateur, un message : *« Ce site ne peut pas fournir de connexion sécurisée »*, avec `ERR_SSL_PROTOCOL_ERROR`. Même erreur sur tous mes domaines, y compris ceux des applications qui n'avaient rien à voir entre elles.

**L'enquête :** Le premier réflexe utile a été de ne pas déboguer application par application. Web, mobile, API et console MinIO tombaient en même temps : le point commun ne pouvait pas être dans leur code, mais dans ce qui reçoit leur trafic HTTPS, c'est-à-dire Traefik. Sur le serveur :

```
docker ps | grep traefik
(aucune sortie)

docker restart dokploy-traefik
Error response from daemon: Cannot restart container dokploy-traefik: failed to set up container networking: ... failed to bind host port 0.0.0.0:80/tcp: address already in use
```

Traefik n'était pas dans la liste des conteneurs en marche, et il refusait de redémarrer parce que quelqu'un d'autre tenait déjà le port 80. Il restait à trouver qui :

```
ss -tlnp | grep -E ':80 |:443 '
LISTEN 0 4096 *:80  *:* users:(("caddy",pid=764,fd=10))
LISTEN 0 4096 *:443 *:* users:(("caddy",pid=764,fd=7))
```

**Le déclic :** Un **Caddy**, installé directement sur le serveur et non dans Docker, occupait les ports 80 et 443. Traefik n'était pas cassé : il était arrêté, et empêché de reprendre sa place. Cela expliquait l'erreur du navigateur : sur le port 443, quelqu'un répondait, mais ce n'était pas le bon, d'où `ERR_SSL_PROTOCOL_ERROR`. Un détail du `docker ps` donne une piste, sans que je l'aie vérifiée : tous les conteneurs, même ceux créés il y a quatre mois, affichaient « Up 10 minutes ». Le serveur ou Docker avait donc redémarré peu avant, et Caddy a très probablement pris les ports avant Traefik.

**La résolution :** J'ai arrêté Caddy, puis relancé Traefik :

```
systemctl stop caddy
docker start dokploy-traefik
```

Nouveau symptôme : le message du navigateur a changé pour `ERR_CONNECTION_TIMED_OUT`. Traefik apparaissait « Up », mais la colonne PORTS de `docker ps` était vide et `ss` ne voyait plus rien sur les ports 80 et 443. Avant d'envisager de réinstaller Dokploy, j'ai vérifié la configuration du conteneur :

```
docker inspect dokploy-traefik --format '{{json .HostConfig.PortBindings}}'
{"443/tcp":[{"HostPort":"443"}],"443/udp":[{"HostPort":"443"}],"80/tcp":[{"HostPort":"80"}]}
```

La configuration était correcte : c'était l'état du conteneur qui était bancal, après un démarrage raté suivi d'un simple `docker start`. Un vrai redémarrage a tout réglé :

```
docker restart dokploy-traefik
```

Les ports 80 et 443 sont réapparus, et le HTTPS est revenu sur mes domaines. Au total, environ une heure de diagnostic.

**Ce que j'en retiens :**
- Quand tout tombe en même temps (web, mobile, API, console MinIO), il ne faut pas déboguer chaque application : il faut chercher ce qu'elles ont en commun. Ici, c'était le reverse proxy.
- Le message d'erreur du navigateur dit où l'on en est. `ERR_SSL_PROTOCOL_ERROR` signifie que quelque chose répond mais que ce n'est pas le bon ; `ERR_CONNECTION_TIMED_OUT` signifie que plus personne n'écoute. Quand l'erreur change, c'est qu'on avance.
- Un `address already in use` se résout en une commande : `ss -tlnp | grep -E ':80 |:443 '` donne le nom du processus qui occupe le port.
- Un conteneur « Up » ne prouve pas qu'il est joignable. Une colonne PORTS vide veut dire qu'aucun port n'est publié. Après un échec de démarrage, un `docker restart` propre vaut mieux qu'un simple `docker start`.
- Avant de réinstaller un outil par réflexe, lire l'état réel : ici, `PortBindings` était correct, donc la réinstallation n'aurait rien réparé.
- Un seul reverse proxy par serveur. Si Caddy ne sert à rien ici, il faut le désactiver au démarrage (`systemctl disable caddy`), sinon il reprendra les ports au prochain redémarrage. Reste à élucider qui l'a installé et pourquoi.
- C'est notre dernier test avant de partir qui nous a sauvés : sans ce réflexe, la panne se serait découverte devant le client. Une mini-liste de vérification (web, `/docs` de l'API, mobile) et une alerte externe de disponibilité, du type UptimeRobot, permettraient de la voir avant tout le monde.
- Le `docker ps` a aussi montré plusieurs ports ouverts sur Internet (PostgreSQL 5432, MinIO 9000-9001, Dokploy 3000, Supavisor 5433 et 6543). Derrière Traefik, ils n'ont pas besoin d'être publiés : c'est un chantier de sécurité à ouvrir.

**Tags :** `#docker` `#dokploy` `#traefik` `#caddy` `#reverse-proxy` `#hostinger` `#ssl` `#réseau` `#agroboom`

---

## Entrée #8 — *(à venir)*

*...*
