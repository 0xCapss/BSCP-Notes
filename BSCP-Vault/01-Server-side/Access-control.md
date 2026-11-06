---
tags: [bscp, server-side, access-control]
niveau: practitioner
statut: en cours
---
# Access control

## En bref
- Le contrôle d'accès consiste à appliquer des contraintes afin de déterminer qui ou quoi est autorisé à effectuer des actions ou à accéder à des ressources.
- Dans le cas du web, il repose sur l'authentification (qui est l'utilisateur) et sur la gestion des sessions (quelles requêtes viennent de ce même utilisateur).
- La gestion des contrôles d'accès est un problème complexe qui impose de nombreuses contraintes. Ces décisions sont très souvent prises par des humains, ce qui augmente encore le risque d'erreur.
- Un contrôle d'accès défaillant a pour impact l'accès non autorisé à des fonctions ou à des données, jusqu'à la prise de contrôle complète de l'application.
- Il se distingue de l'authentification : l'authentification établit qui est l'utilisateur ([[Authentication]]), le contrôle d'accès décide ce qu'il a le droit de faire.

![](Access-control.png)

## Modèles de contrôle d'accès
- Contrôle vertical : des rôles différents ont accès à des fonctions différentes (utilisateur standard, administrateur). Sa défaillance donne l'escalade verticale de privilèges (vertical privilege escalation).
- Contrôle horizontal : des utilisateurs de même niveau n'ont accès qu'à leurs propres ressources. Sa défaillance donne l'escalade horizontale de privilèges (horizontal privilege escalation).
- Contrôle dépendant du contexte : l'accès dépend de l'état de l'application ou de l'ordre des étapes (par exemple une confirmation qui ne doit être accessible qu'après une étape précédente). Sa défaillance permet de sauter ou de rejouer une étape.

Mécanismes de décision, utiles pour comprendre pourquoi certains contournements existent :
- Par rôles : des droits rattachés à des groupes de comptes.
- Par listes de contrôle d'accès : des droits rattachés à chaque ressource.
- Par attributs : une règle évalue des propriétés de l'utilisateur, de la ressource et du contexte.
- Programmatique (dans le code, requête par requête) ou déclaratif (règles de configuration appliquées par un composant en amont, comme un pare-feu applicatif ou un proxy). Quand deux composants appliquent des règles différentes à la même requête, les contournements de plateforme deviennent possibles.

## Types et variantes
- Fonctionnalité non protégée : une ressource ou fonction sensible est accessible sans aucun contrôle dès qu'on en connaît (ou devine) l'URL, indépendamment du rôle de l'utilisateur.
- Contrôle d'accès basé sur un paramètre utilisateur : le rôle ou les droits sont stockés dans un élément modifiable côté client (paramètre de requête, cookie, champ caché) plutôt que dérivés côté serveur. L'ajout d'un champ protégé dans le corps d'une requête de mise à jour relève de l'affectation de masse (mass assignment).
- Escalade verticale de privilèges : un utilisateur avec des droits non administrateurs parvient à accéder à des fonctions réservées à un rôle supérieur.
- Escalade horizontale de privilèges : un utilisateur accède aux ressources d'un autre utilisateur de même niveau de privilège.
- Référence directe non sécurisée à un objet (IDOR, insecure direct object reference) : cas particulier de l'escalade horizontale où l'application utilise directement un identifiant fourni par l'utilisateur (nom, numéro, nom de fichier) pour accéder à un objet, sans contrôler qu'il lui appartient.
- Escalade horizontale vers verticale : si la ressource atteinte appartient à un compte administrateur, l'escalade horizontale devient verticale.

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Fonctionnalité non protégée | `/admin`, `robots.txt`, URL dans le JS | [[Access-control-fonctionnalite-non-protegee]] | Apprentice |
| Paramètre contrôlé par l'utilisateur | `Admin=true`, `roleid` | [[Access-control-parametre-utilisateur]] | Apprentice |
| Escalade horizontale | `?id=carlos`, GUID, fuites | [[Access-control-escalade-horizontale]] | Apprentice |
| IDOR | `/download-transcript/1.txt` | [[Access-control-idor]] | Apprentice |
| Contournement lié à la plateforme | `X-Original-URL`, changement de méthode | [[Access-control-contournement-plateforme]] | Practitioner |
| Multi-étapes et `Referer` | étape de confirmation, en-tête `Referer` | [[Access-control-processus-multi-etapes-et-referer]] | Practitioner |

## Méthodologie
Le détail de chaque type se trouve dans les notes du tableau ci-dessus.

### Démarche générale
1. Cartographier les fonctions sensibles et les rôles (non authentifié, utilisateur standard, administrateur). Repérer systématiquement : `robots.txt`, `sitemap.xml`, JavaScript, commentaires HTML, panneaux d'administration, réponses JSON portant un champ de rôle, cookies et en-têtes reçus après connexion avec chaque compte fourni.
2. Rejouer chaque requête sensible dans trois contextes : sans session, avec une session de faible privilège, avec une session d'administrateur. Une extension de Burp qui rejoue le trafic d'un compte avec la session d'un autre (Autorize, AuthMatrix) automatise cette comparaison.
3. Tester l'action elle-même, pas seulement l'affichage de la page : une page peut s'afficher sans que l'action soit autorisée, et inversement.
4. Pour chaque méthode HTTP exposée par un point d'entrée, noter ce qui change : 405 (méthode non gérée), 401 ou 403 (refus), 200 avec un contenu différent. Un refus sur une méthode n'implique pas un refus sur les autres.
5. Établir la preuve : l'accès réel à la donnée ou à la fonction d'un autre compte, pas seulement un changement de code de statut.

Signaux de réussite ou d'échec à comparer entre contextes : code de statut, longueur de réponse, corps d'une redirection non vide, message d'erreur différent. Des réponses uniformes (même page pour tous les refus) peuvent masquer un écart, et la longueur seule ne suffit pas.

### Fonctionnalité non protégée
- Lire `/robots.txt` et `/sitemap.xml` : chemins d'administration listés dans `Disallow`. Limite : fichier vide ou leurre.
- Rechercher dans le JavaScript et les commentaires HTML les chemins contenant `admin`, `internal` ou `manage`, y compris dans une affectation d'attribut ou une route. Limite : fichiers minifiés, donc recherche en texte intégral.
- Énumérer les chemins courants (`/admin`, `/administrator`, `/manage`, `/console`) avec un outil de découverte de contenu : un 200 ou un 403 là où l'on attend un 404 est un signal. Limite : bruit et limitation de débit.
- Signal d'échec : redirection vers la page de connexion ou 403 pour tous les contextes.

### Paramètre contrôlé par l'utilisateur et affectation de masse
- Cookie en clair portant un drapeau (de type `Admin=false` vers `Admin=true`), paramètre de requête ou champ caché (`admin=true`, `role=admin`, `isAdmin=1`). Signal : page ou fonction d'administration accessible. Limite : cookie signé ou chiffré, ou nom de paramètre inexact.
- Ajout d'un champ de rôle (`"roleid":2`, `"role":"admin"`) dans le corps JSON d'une mise à jour de profil, surtout quand la réponse JSON montre déjà ce champ. Signal : champ accepté et droits changés. Limite : le serveur ignore les champs inconnus ou applique une liste blanche de champs.
- Changer le format de contenu (JSON vers formulaire, ou l'inverse) ou dupliquer un paramètre pour contourner un filtre de champs. Limite : dépend de l'application.
- Regarder aussi les fonctions de changement d'e-mail, de mot de passe ou de profil comme points d'entrée de champs cachés, y compris pour agir sur le compte d'un autre utilisateur.
- Décoder tout jeton de session (base64, JWT) à la recherche d'un rôle lisible. Limite : la signature empêche la modification ; voir [[JWT-attacks]].

### Escalade horizontale et référence directe non sécurisée
- Remplacer un identifiant lisible par celui d'un autre compte. Signal : données de l'autre utilisateur. Limite : contrôle de propriété côté serveur.
- Incrémenter ou décrémenter un identifiant numérique ou un nom de fichier séquentiel (`1.txt`, `2.txt`). Un compte qui génère l'objet aide à comprendre le schéma. Pour une série d'identifiants, comparer les longueurs de réponse avec Intruder (limitation de débit possible).
- Identifiants imprévisibles (GUID, globally unique identifier) : chercher celui de la cible dans les pages publiques (articles, commentaires, profils) ou dans les exports et notifications. Impasse si l'identifiant n'est exposé nulle part.
- Tester l'identifiant à tous les endroits possibles : chemin, chaîne de requête, corps, en-têtes, cookies (surtout pour les API).
- Lire le corps des redirections : une réponse 302 vers la page de connexion peut porter la donnée demandée dans son corps. Le navigateur masque ce corps, il faut un proxy comme Burp. Se rattache à [[Information-disclosure]].
- Quand l'identifiant désigne un compte administrateur, chercher les champs sensibles dans le HTML (mot de passe pré-rempli, valeur masquée visuellement mais présente dans le source), ce qui fait basculer l'escalade en verticale.
- API : propriétés d'objet exposées en lecture puis renvoyées en écriture, listes d'objets sans filtre (REST ou [[GraphQL-API]]).

### Contournement lié à la plateforme (techniques génériques)
Mécanisme : un composant en amont (pare-feu, proxy, front-end) applique la règle sur la ligne de requête, alors que le serveur d'application peut router selon un en-tête de réécriture ou normaliser le chemin autrement.
- En-têtes de réécriture d'URL (`X-Original-URL`, `X-Rewrite-URL`) avec un chemin autre que celui de la ligne de requête. Reconnaissance : fournir un chemin inexistant et observer une réponse « not found » venant de l'application, ce qui prouve seulement la prise en compte de l'en-tête, pas l'accès. Limite : dépend du framework ou du proxy ; les paramètres de requête peuvent suivre une gestion différente.
- Changement de méthode (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`) quand la règle est liée à une méthode. Signal : passage de 401 ou 403 à 200. Limite : la méthode alternative peut exiger un autre format de paramètres.
- Surcharge de méthode (`X-HTTP-Method-Override: PUT`, ou paramètre `_method`) sur une requête `POST`. Limite : souvent désactivée.
- Variantes de chemin pour une règle à correspondance exacte : casse (`/Admin`), barre oblique finale (`/admin/`), doubles barres, extension (`/admin.json`), paramètre de chemin avec point-virgule (`/admin;x`), encodage (`/%61dmin`), segment `/./`. Limite : dépend de la normalisation.
- En-têtes de transfert (`X-Forwarded-For: 127.0.0.1`) face à une règle fondée sur l'adresse source. Limite : ignorés si le proxy n'est pas configuré pour les faire confiance.

### Processus en plusieurs étapes et Referer (techniques génériques)
Rattaché au contrôle dépendant du contexte et à [[Business-logic]].
- Rejouer directement une étape tardive en omettant les précédentes : fonctionne quand seule la première étape est protégée. Limite : jetons d'état propres à chaque étape.
- Rejouer chaque étape sensible avec la session d'un compte moins privilégié pour savoir quelles étapes vérifient le rôle (deux sessions distinctes).
- Présence, absence et modification de l'en-tête `Referer` quand le contrôle est fondé sur la provenance. Signal : refus sans l'en-tête, succès avec une valeur attendue. Limite : le serveur peut contrôler l'origine complète. L'en-tête étant fourni par le client, il ne prouve rien.

Les éléments courts réutilisables de ces familles sont regroupés dans [[Payloads-cheatsheet]].

## Pièges et points d'attention BSCP
- Rôle de cette classe dans l'examen : le contrôle d'accès sert typiquement d'accès initial à l'administration ou de pivot vers un compte privilégié, avant la suite de la chaîne. Le temps étant limité, commencer par la cartographie et les tests à faible coût (trois contextes de session, identifiants, en-têtes).
- Ne pas se contenter d'un seul rôle : comparer systématiquement non authentifié, utilisateur standard et administrateur pour chaque fonction sensible.
- Une fonctionnalité peut être invisible dans l'interface mais rester accessible côté serveur : l'absence de lien n'est pas une preuve d'absence de vulnérabilité. Inversement, une URL difficile à deviner n'est pas un contrôle.
- Un identifiant non prévisible (GUID) ne signifie pas que le contrôle est correct : s'il est exposé ailleurs (export, notification, historique, article), l'escalade horizontale reste exploitable.
- Ne pas s'arrêter à la première preuve d'escalade horizontale : mesurer l'étendue réelle (compte administrateur, champs sensibles) et confirmer que la donnée d'un autre compte est bien obtenue.
- Lire le corps des redirections et des messages d'erreur : un code 302 ou 403 peut s'accompagner de la donnée.
- Ne pas confondre absence de contrôle d'accès et défaut d'authentification (voir [[Authentication]]).
- Éléments qui bloquent l'exploitation et faussent un test négatif : jeton anti-falsification, liaison de la session, limitation de débit, contrôles asynchrones, réponses uniformes. Les noter avant de conclure qu'il n'y a pas de faille.
- Un contrôle appliqué par un composant en amont se contourne parfois par réécriture d'URL, autre méthode ou variante de chemin (voir la méthodologie).

## Prévention
- Appliquer le contrôle d'accès côté serveur, sur chaque requête, jamais uniquement côté client (masquage dans l'interface ou en JavaScript).
- Centraliser la logique d'autorisation dans un seul mécanisme central plutôt que de la dupliquer dans chaque contrôleur, pour éviter les oublis.
- Refuser par défaut (deny by default) et n'autoriser explicitement que ce qui est nécessaire, plutôt que de bloquer une liste de cas connus.
- Ne jamais stocker le rôle ou les droits dans un élément contrôlable par le client (paramètre, cookie, champ caché) ; les dériver côté serveur à partir de la session authentifiée.
- Ne pas se fier à des URLs non devinables comme seule protection (sécurité par l'obscurité) ; appliquer un contrôle d'accès explicite sur chaque endpoint sensible.
- Utiliser des identifiants non séquentiels (GUID) en défense en profondeur, sans que cela remplace une vérification explicite que la ressource demandée appartient bien à l'utilisateur courant. Faire cette vérification de propriété jusqu'à la couche de données.
- Ne pas renvoyer de données dans le corps d'une redirection ni d'un message d'erreur, et ne pas pré-remplir de mot de passe dans une page.
- Désactiver côté serveur d'application les en-têtes de réécriture d'URL et de surcharge de méthode qui ne sont pas nécessaires, et normaliser le chemin avant d'appliquer les règles.
- Ne pas faire confiance à `Referer` ni aux en-têtes de transfert fournis par le client.
- Contrôler le rôle à chaque étape d'un processus en plusieurs étapes, pas seulement à la première.
- Utiliser une liste blanche de champs modifiables pour éviter l'affectation de masse.
- Journaliser les refus d'accès et alerter sur les séries anormales ; écrire des tests automatisés d'autorisation (chaque rôle contre chaque fonction sensible).

## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-17
- [ ] Practitioner
- [ ] Expert

## Journal des labs

> [!warning] Étapes rédigées de mémoire
> Cette section était vide dans la page d'origine. Les étapes ci-dessous sont reconstituées de mémoire à partir des solutions publiques de PortSwigger et n'ont pas été rejouées : à vérifier sur chaque lab. Compte standard des labs : `wiener:peter`. Compte administrateur (labs 11 à 13) : `administrator:admin`.

### Lab 1 - Fonctionnalité d'administration non protégée
*Unprotected admin functionality* - Apprentice - note : [[Access-control-fonctionnalite-non-protegee]]

Ce lab possède un panneau d'administration non protégé. Supprimez l'utilisateur `carlos`.

1. Allez sur `/robots.txt` : le fichier révèle le chemin du panneau d'administration.
2. Chargez ce chemin dans le navigateur et supprimez `carlos`.

### Lab 2 - Fonctionnalité d'administration non protégée avec URL imprévisible
*Unprotected admin functionality with unpredictable URL* - Apprentice - note : [[Access-control-fonctionnalite-non-protegee]]

Le panneau d'administration est à une URL imprévisible, mais divulguée quelque part dans l'application. Supprimez `carlos`.

1. Affichez le code source de la page d'accueil (`Ctrl+U`) ou examinez la réponse dans Burp.
2. Repérez dans le JavaScript l'URL du panneau d'administration (`adminPanelTag.setAttribute('href', ...)`).
3. Chargez cette URL et supprimez `carlos`.

### Lab 3 - Rôle de l'utilisateur contrôlé par un paramètre de requête
*User role controlled by request parameter* - Apprentice - note : [[Access-control-parametre-utilisateur]]

Le panneau d'administration est à `/admin` et détermine l'accès à partir d'un cookie forgeable. Supprimez `carlos`.

1. Connectez-vous avec `wiener:peter` avec le proxy Burp actif et constatez le cookie `Admin=false`.
2. Chargez `/admin` en interceptant la requête et remplacez la valeur par `Admin=true`.
3. Dans la page d'administration, repérez le lien de suppression de `carlos` et rejouez-le avec `Admin=true`.

### Lab 4 - Rôle de l'utilisateur modifiable dans le profil
*User role can be modified in user profile* - Apprentice - note : [[Access-control-parametre-utilisateur]]

Le panneau d'administration est à `/admin` et n'est accessible qu'avec un `roleid` de 2. Supprimez `carlos`.

1. Connectez-vous avec `wiener:peter` et changez votre adresse e-mail. Dans Burp, constatez que la réponse JSON contient un champ `roleid`.
2. Envoyez la requête de changement d'e-mail dans Repeater et ajoutez `"roleid":2` dans le corps JSON. Envoyez-la.
3. Chargez `/admin` et supprimez `carlos`.

### Lab 5 - Identifiant d'utilisateur contrôlé par un paramètre de requête
*User ID controlled by request parameter* - Apprentice - note : [[Access-control-escalade-horizontale]]

Le lab a une escalade horizontale sur la page de compte. Récupérez la clé d'API de `carlos` et soumettez-la.

1. Connectez-vous avec `wiener:peter` et ouvrez « My account » : l'URL contient `?id=wiener`.
2. Remplacez `id=wiener` par `id=carlos`.
3. Copiez la clé d'API affichée et soumettez-la avec « Submit solution ».

### Lab 6 - Identifiant d'utilisateur contrôlé par un paramètre, avec identifiants imprévisibles
*User ID controlled by request parameter, with unpredictable user IDs* - Apprentice - note : [[Access-control-escalade-horizontale]]

Les utilisateurs sont identifiés par des GUID. Récupérez la clé d'API de `carlos`.

1. Trouvez un article de blog écrit par `carlos` et cliquez sur son nom : l'URL contient son `userId` (GUID).
2. Connectez-vous avec `wiener:peter`, ouvrez « My account » et remplacez la valeur de `id` par le GUID de `carlos`.
3. Copiez la clé d'API affichée et soumettez-la.

### Lab 7 - Identifiant d'utilisateur contrôlé par un paramètre, avec fuite de données dans une redirection
*User ID controlled by request parameter with data leakage in redirect* - Apprentice - note : [[Access-control-escalade-horizontale]]

Le lab fuit des informations sensibles dans le corps d'une redirection. Récupérez la clé d'API de `carlos`.

1. Connectez-vous avec `wiener:peter`, ouvrez « My account » et envoyez la requête dans Repeater.
2. Remplacez `id=wiener` par `id=carlos` et envoyez la requête.
3. La réponse est une redirection (`302`) vers la page de connexion, mais son corps contient la clé d'API de `carlos`. Copiez-la et soumettez-la.

### Lab 8 - Identifiant d'utilisateur contrôlé par un paramètre, avec divulgation du mot de passe
*User ID controlled by request parameter with password disclosure* - Apprentice - note : [[Access-control-escalade-horizontale]]

La page de compte pré-remplit le champ du mot de passe. Récupérez le mot de passe de `administrator`, connectez-vous et supprimez `carlos`.

1. Connectez-vous avec `wiener:peter` et ouvrez « My account ».
2. Remplacez `id=wiener` par `id=administrator`.
3. Dans la réponse, repérez la valeur du champ de mot de passe masqué (dans Burp ou avec l'inspecteur du navigateur).
4. Connectez-vous avec `administrator` et ce mot de passe, ouvrez `/admin` et supprimez `carlos`.

### Lab 9 - Références directes non sécurisées à un objet
*Insecure direct object references* - Apprentice - note : [[Access-control-idor]]

Le lab stocke les transcriptions du chat en fichiers statiques. Trouvez le mot de passe de `carlos` et connectez-vous.

1. Ouvrez l'onglet « Live chat », envoyez un message puis cliquez sur « View transcript ».
2. Constatez que le fichier est servi sous un nom incrémental, par exemple `/download-transcript/2.txt`.
3. Remplacez le numéro par `1.txt` : la transcription contient le mot de passe de `carlos`.
4. Connectez-vous en tant que `carlos` avec ce mot de passe.

### Lab 10 - Contrôle d'accès par URL contournable
*URL-based access control can be circumvented* - Practitioner - note : [[Access-control-contournement-plateforme]]

Le site bloque l'accès externe à `/admin` au niveau du front-end, mais l'application accepte l'en-tête `X-Original-URL`. Supprimez `carlos`.

1. Chargez `/admin` et constatez que l'accès est refusé par le front-end.
2. Envoyez une requête `GET /` dans Repeater avec l'en-tête `X-Original-URL: /invalid` : l'application répond « not found », ce qui prouve que l'en-tête est pris en compte.
3. Remplacez par `X-Original-URL: /admin` : le panneau d'administration s'affiche.
4. Pour supprimer l'utilisateur, envoyez `GET /?username=carlos` avec `X-Original-URL: /admin/delete`.

### Lab 11 - Contrôle d'accès par méthode contournable
*Method-based access control can be circumvented* - Practitioner - note : [[Access-control-contournement-plateforme]]

Le contrôle d'accès dépend en partie de la méthode HTTP. Devenez administrateur avec `wiener:peter`.

1. Connectez-vous en tant qu'administrateur (`administrator:admin`), ouvrez le panneau d'administration et promouvez `carlos` : repérez la requête `POST /admin-roles` (`username=carlos&action=upgrade`) et envoyez-la dans Repeater.
2. Ouvrez une session privée, connectez-vous avec `wiener:peter` et copiez le cookie de session.
3. Dans Repeater, remplacez le cookie de session par celui de `wiener` : la requête est refusée.
4. Changez la méthode en `GET` (clic droit, « Change request method »), remplacez le nom d'utilisateur par `wiener` et envoyez la requête.

### Lab 12 - Processus en plusieurs étapes sans contrôle d'accès sur une étape
*Multi-step process with no access control on one step* - Practitioner - note : [[Access-control-processus-multi-etapes-et-referer]]

Le changement de rôle se fait en plusieurs étapes et l'une d'elles n'est pas protégée. Devenez administrateur avec `wiener:peter`.

1. Connectez-vous en tant qu'administrateur (`administrator:admin`), promouvez `carlos` et observez la requête de confirmation (`POST /admin-roles` avec `action=upgrade&confirmed=true&username=carlos`).
2. Envoyez cette requête de confirmation dans Repeater.
3. Ouvrez une session privée, connectez-vous avec `wiener:peter` et copiez son cookie de session.
4. Dans Repeater, collez le cookie de `wiener`, remplacez le nom d'utilisateur par `wiener` et envoyez la requête de confirmation.

### Lab 13 - Contrôle d'accès basé sur l'en-tête Referer
*Referer-based access control* - Practitioner - note : [[Access-control-processus-multi-etapes-et-referer]]

Le contrôle d'accès de certaines fonctions d'administration repose sur l'en-tête `Referer`. Devenez administrateur avec `wiener:peter`.

1. Connectez-vous en tant qu'administrateur (`administrator:admin`), promouvez `carlos` et envoyez dans Repeater la requête `GET /admin-roles?username=carlos&action=upgrade`.
2. Ouvrez une session privée, connectez-vous avec `wiener:peter` et copiez son cookie de session.
3. Dans Repeater, collez le cookie de `wiener`, remplacez le nom d'utilisateur par `wiener` et conservez l'en-tête `Referer` du panneau d'administration. Envoyez la requête.
4. Constatez que sans cet en-tête la requête est refusée : c'est lui qui sert de contrôle.
