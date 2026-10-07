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

![](Access-control.png)

## Types et variantes
- Fonctionnalité non protégée : une ressource ou fonction sensible est accessible sans aucun contrôle dès qu'on en connaît (ou devine) l'URL, indépendamment du rôle de l'utilisateur.
- Contrôle d'accès basé sur un paramètre utilisateur : le rôle ou les droits sont stockés dans un élément modifiable côté client (paramètre de requête, cookie, champ caché) plutôt que dérivés côté serveur.
- Escalade verticale de privilèges (vertical privilege escalation) : un utilisateur avec des droits non administrateurs parvient à accéder à des fonctions réservées à un rôle supérieur.
- Escalade horizontale de privilèges (horizontal privilege escalation) : un utilisateur accède aux ressources d'un autre utilisateur de même niveau de privilège, très proche de l'IDOR.
- Escalade horizontale vers verticale : un vecteur horizontal peut parfois être détourné pour atteindre un compte à privilèges plus élevés (par exemple un compte administrateur), ce qui transforme l'escalade horizontale en escalade verticale.

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Fonctionnalité non protégée | `/admin`, `robots.txt`, URL dans le JS | [[Access-control-fonctionnalite-non-protegee]] | Apprentice |
| Paramètre contrôlé par l'utilisateur | `Admin=true`, `roleid` | [[Access-control-parametre-utilisateur]] | Apprentice |
| Escalade horizontale | `?id=carlos`, GUID, fuites | [[Access-control-escalade-horizontale]] | Apprentice |
| IDOR | `/download-transcript/1.txt` | [[Access-control-idor]] | Apprentice |
| Contournement lié à la plateforme | `X-Original-URL`, changement de méthode | [[Access-control-contournement-plateforme]] | Practitioner |
| Multi-étapes et `Referer` | étape de confirmation, en-tête `Referer` | [[Access-control-processus-multi-etapes-et-referer]] | Practitioner |

## Comment détecter
- Cartographier toutes les fonctions sensibles ou administratives de l'application (menu, JavaScript, sitemap Burp) puis tenter d'y accéder directement en tant qu'utilisateur non privilégié ou non authentifié.
- Comparer le comportement entre plusieurs comptes de rôles différents (matrice de droits) pour repérer les incohérences.
- Rechercher les URLs sensibles référencées dans le code JavaScript, les commentaires HTML, `robots.txt`, `sitemap.xml`, ou déductibles par un pattern prévisible.
- Tester la modification de tout paramètre, cookie ou champ caché qui ressemble à un rôle ou un identifiant (`role`, `admin`, `id`, `uid`, `account`) en changeant sa valeur.
- Pour l'escalade horizontale, tester le changement d'identifiant de ressource (numérique incrémental ou deviné) avec un compte différent, sur toutes les fonctionnalités qui manipulent des ressources utilisateur (profil, factures, messages).
- Vérifier le contrôle d'accès sur toutes les méthodes HTTP exposées par un endpoint (GET, POST, PUT, DELETE), pas seulement celle utilisée par l'interface normale.

## Comment exploiter (principe)
1. Cartographier les fonctions sensibles et les rôles (non authentifié, utilisateur, administrateur).
2. Rejouer chaque requête sensible avec une session de moindre privilège, sans session, et avec un identifiant modifié.
3. Chercher les informations qui fuient : URL cachées dans le JavaScript et `robots.txt`, identifiants dans les articles, corps des redirections.
4. Si un front-end bloque l'accès, essayer en-têtes de réécriture d'URL, autres méthodes HTTP, variantes de chemin.
5. Pour les actions en plusieurs étapes, rejouer directement la dernière étape.

Détail par type dans les notes de la table ci-dessus.

## Pièges et points d'attention BSCP
- Ne pas se contenter de tester un seul rôle : comparer systématiquement au moins trois cas (non authentifié, utilisateur standard, utilisateur admin) pour chaque fonction sensible.
- Une fonctionnalité peut être invisible dans l'interface mais rester accessible côté serveur : l'absence de lien n'est pas une preuve d'absence de vulnérabilité.
- Un identifiant non prévisible (GUID) ne signifie pas que le contrôle d'accès est correct : si l'ID est exposé ailleurs (export, notification, historique, requête précédente), l'escalade horizontale reste exploitable.
- Vérifier le contrôle d'accès sur toutes les méthodes HTTP d'un même endpoint (GET, POST, PUT, DELETE), pas seulement celle utilisée par l'UI normale.
- Une escalade horizontale peut devenir verticale si la ressource accédée appartient à un compte administrateur : ne pas s'arrêter à la première preuve d'IDOR, vérifier l'étendue réelle.
- Ne pas confondre absence de contrôle d'accès et défaut d'authentification : bien distinguer "qui es-tu" (authentification) de "as-tu le droit" (autorisation).
- Un contrôle appliqué par un composant en amont (pare-feu, proxy) peut être contourné par un en-tête de réécriture d'URL ou une autre méthode HTTP.
- L'en-tête `Referer` est contrôlé par le client : il ne prouve rien.

## Prévention
- Appliquer le contrôle d'accès côté serveur, sur chaque requête, jamais uniquement côté client (masquage dans l'interface ou en JavaScript).
- Centraliser la logique d'autorisation dans un seul mécanisme vérifié plutôt que de la dupliquer dans chaque contrôleur, pour éviter les oublis.
- Refuser par défaut (deny by default) et n'autoriser explicitement que ce qui est nécessaire, plutôt que de bloquer une liste de cas connus.
- Ne jamais stocker le rôle ou les droits dans un élément contrôlable par le client (paramètre, cookie, champ caché) ; les dériver côté serveur à partir de la session authentifiée.
- Ne pas se fier à des URLs non devinables comme seule protection (sécurité par l'obscurité) ; appliquer un contrôle d'accès explicite sur chaque endpoint sensible.
- Utiliser des identifiants non séquentiels (GUID) en défense en profondeur, sans que cela remplace une vérification explicite que la ressource demandée appartient bien à l'utilisateur courant.

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

## Mes notes
- 

## Liens
- [[Access-control-fonctionnalite-non-protegee]]
- [[Access-control-parametre-utilisateur]]
- [[Access-control-escalade-horizontale]]
- [[Access-control-idor]]
- [[Access-control-contournement-plateforme]]
- [[Access-control-processus-multi-etapes-et-referer]]
- [[Authentication]]
- [[Business-logic]]
- [[Information-disclosure]]
- [[Path-traversal]]
- [[Payloads-cheatsheet]]
