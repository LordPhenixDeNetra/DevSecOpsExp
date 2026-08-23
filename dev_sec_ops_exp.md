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

## Entrée #2 — *(à venir)*

*...*