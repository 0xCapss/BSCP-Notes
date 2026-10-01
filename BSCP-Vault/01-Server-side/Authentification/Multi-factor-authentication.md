---
tags: [bscp, server-side, authentication, mfa]
niveau: apprentice
statut: en cours
---
# Multi-factor authentication

## En bref
- Méthode de sécurité qui demande à un utilisateur de fournir plusieurs preuves de son identité, en combinant des facteurs de catégories différentes (connaissance, possession, inhérent), pour accéder à un compte ou une application.
- À distinguer de la "two-step verification" (deux étapes mais avec des facteurs de la même catégorie, par exemple un mot de passe puis une réponse à une question secrète) : ce n'est pas la même robustesse, même si les deux ajoutent une étape de connexion.
- Un défaut d'implémentation du second facteur peut annuler tout l'intérêt de la mesure et ramener la sécurité au niveau d'une authentification par mot de passe seul.

## Types et variantes
- Facteur de possession matérialisé par :
	- Une application d'authentification générant un OTP (One-Time Password), par exemple Microsoft/Google Authenticator.
	- Un token matériel dédié (type Yubikey).
	- Un code reçu par SMS, vulnérable au SIM swapping (prise de contrôle de la carte SIM de la victime) et à l'interception par phishing en temps réel (proxy transparent type Evilginx qui relaie le code à la volée).
	- Une notification push à valider sur un appareil de confiance.
- Mécanismes de secours (backup/recovery codes) : généralement affichés une seule fois à la configuration, parfois réutilisables ou non invalidés après usage, souvent moins protégés que l'OTP principal (pas de limitation de tentatives sur leur saisie, format prévisible).
- Cookie "se souvenir de cet appareil" (trusted device) : permet de sauter la demande de second facteur sur les connexions suivantes depuis le même navigateur ; sa sécurité dépend entièrement de la robustesse de la valeur (imprévisible, signée, liée au compte et à l'appareil).

## Comment détecter
- Vérifier si des pages ou fonctions censées être accessibles uniquement après validation complète (étape 1 + étape 2) répondent normalement en y naviguant directement juste après l'étape 1, sans jamais soumettre le second facteur.
- Vérifier si l'identité du compte visé par la deuxième étape est portée par une valeur modifiable côté client (cookie, paramètre caché, champ de formulaire) plutôt que dérivée d'un état de session vérifié côté serveur.
- Déterminer l'espace de recherche du code de vérification (nombre de chiffres) et tester s'il existe une limitation de tentatives : sans elle, le code devient brute-forçable.
- Vérifier si un code de vérification reste valide après un usage réussi (pas d'invalidation immédiate), ou au-delà d'une fenêtre de temps raisonnable (expiration).
- Si la vérification semble se faire côté client (JS) ou renvoie un indicateur de statut explicite, intercepter la réponse pour voir si un champ comme `"verified":false` est manipulable avant d'être interprété par l'application.
- Tester si un cookie "se souvenir de cet appareil" existe et s'il est prévisible, rejouable sur un autre navigateur, ou non lié de façon vérifiable au compte.
- Vérifier que les autres flux sensibles du compte (réinitialisation de mot de passe, changement d'email, API mobile parallèle) imposent eux aussi la validation du second facteur, et n'offrent pas un chemin parallèle qui le contourne.

## Comment exploiter (principe)
### Accès direct après la première étape (bypass simple)
- Si l'application considère l'utilisateur comme "quasi connecté" dès la validation du mot de passe, avant la validation du second facteur, il est parfois possible d'accéder directement aux pages ou fonctions réservées aux utilisateurs pleinement authentifiés en forçant la navigation vers leur URL, sans jamais soumettre de code de vérification.

### Logique défaillante de liaison entre les deux étapes
- Le site ne vérifie pas correctement, une fois la première étape terminée, que c'est bien le même utilisateur qui effectue la seconde. Ce défaut peut reposer sur un cookie, mais aussi sur un paramètre caché, un champ de formulaire ou un état de session mal vérifié côté serveur : le point commun est l'absence de revérification de l'identité entre les deux étapes, quel que soit le support utilisé.
- Exemple avec un cookie : l'utilisateur se connecte avec ses propres identifiants valides lors de la première étape :
```
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com
...
username=carlos&password=qwerty
```
- Un cookie associé à son compte lui est alors attribué, avant qu'il ne soit redirigé vers la deuxième étape du processus de connexion :
```
HTTP/1.1 200 OK

Set-Cookie: account=carlos


GET /login-steps/second HTTP/1.1

Cookie: account=carlos
```
- Lors de l'envoi du code de vérification, la requête utilise ce cookie pour déterminer à quel compte l'utilisateur tente d'accéder :
```
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=carlos
...
verification-code=123456
```
- Un attaquant disposant de ses propres identifiants valides peut se connecter normalement à l'étape 1, puis modifier la valeur du cookie `account` pour y placer le nom de la victime avant de soumettre l'étape 2 :
```
POST /login-steps/second HTTP/1.1
Host: vulnerable-website.com
Cookie: account=victim-user
...
verification-code=123456
```
- Cela reste insuffisant seul : il faut encore connaître ou deviner le code de vérification de la victime. L'attaquant a besoin de ses propres identifiants valides pour franchir l'étape 1, mais jamais du mot de passe de la victime. C'est en combinant ce défaut avec un brute force du code que l'attaque devient critique (voir ci-dessous).

### Brute force du code de vérification
- Un code de vérification est souvent un nombre à 4 ou 6 chiffres, soit un espace de recherche de 10 000 à 1 000 000 de valeurs : praticable avec Burp Intruder (attaque Sniper, position sur le code, type de payload Numbers) si rien d'autre ne limite les tentatives.
- Condition indispensable à cette attaque : l'absence de limitation de tentatives sur la saisie du code. Sans cette faiblesse, le brute force est impraticable (voir [[Authentication]] pour la logique de rate limiting déjà vue sur les mots de passe).
- Burp Intruder standard peut être trop lent pour couvrir tout l'espace en conditions d'examen ; Turbo Intruder est souvent nécessaire pour une attaque praticable dans le temps imparti.
- Vérifier aussi si un code reste valide après un usage réussi (pas d'invalidation) ou au-delà d'une fenêtre de temps large (absence d'expiration) : deux faiblesses distinctes de la limitation de tentatives, qui élargissent la fenêtre d'exploitation.

### Manipulation de la réponse de vérification
- Quand la décision "code valide ou non" repose sur une valeur renvoyée au client plutôt que strictement vérifiée et appliquée côté serveur, intercepter la réponse et modifier l'indicateur de statut (par exemple un corps JSON `"verified":false` changé en `true`) peut suffire à obtenir l'accès sans connaître le bon code. Faille de confiance dans la réponse, à bien distinguer de la logique défaillante entre étapes vue plus haut.

### Contournement via les mécanismes de secours ou de confiance
- Les codes de secours (backup codes) contournent entièrement le second facteur principal ; s'ils ne sont pas soumis aux mêmes protections (limitation de tentatives, format suffisamment long), ils deviennent le maillon faible à cibler plutôt que l'OTP principal.
- Un cookie "se souvenir de cet appareil" prévisible, non signé, ou rejouable sur un autre navigateur permet de sauter complètement la demande de second facteur sur les connexions suivantes.
- Vérifier si d'autres flux du compte (réinitialisation de mot de passe, changement d'email, API mobile) imposent eux aussi le second facteur : un chemin parallèle qui ne le fait pas revient à contourner toute la protection (voir [[Autres vulnérabilité dans les mécanismes d'authentification]]).

## Pièges et points d'attention BSCP
- Toujours distinguer les deux causes qui peuvent se combiner dans un même scénario : l'absence de limitation de tentatives sur le code, et la logique défaillante de liaison d'identité entre les deux étapes. Ce sont deux vulnérabilités séparées, même quand l'exploitation les enchaîne.
- L'absence de limitation de tentatives et l'absence d'expiration du code sont deux contrôles différents : un code non limité en tentatives mais qui expire vite reste praticable, un code illimité dans le temps mais limité en tentatives beaucoup moins.
- Ne pas se limiter au cookie comme seul support possible de la faille de liaison entre étapes : tester aussi les paramètres cachés et l'état de session côté serveur.
- Un faux "MFA" qui combine deux facteurs de la même catégorie (deux facteurs de connaissance, par exemple) n'est qu'une "two-step verification" : la distinction compte pour l'analyse de robustesse, pas seulement pour le vocabulaire.

## Prévention
- Ne jamais considérer la session comme authentifiée tant que toutes les étapes requises n'ont pas été validées côté serveur ; refuser l'accès à toute ressource protégée tant que le second facteur n'est pas confirmé.
- Dériver l'identité du compte en cours de vérification depuis un état de session signé et vérifié côté serveur, jamais depuis une valeur modifiable par le client (cookie, paramètre, champ caché).
- Appliquer une limitation stricte des tentatives sur le code de vérification et sur les codes de secours, avec verrouillage ou délai croissant après quelques échecs.
- Faire expirer les codes de vérification rapidement et les invalider immédiatement après un usage réussi, qu'il soit valide ou non.
- Ne jamais faire reposer la décision finale de validation sur une donnée renvoyée au client ; la vérification et la décision d'accès doivent être strictement côté serveur.
- Protéger les mécanismes de secours (backup codes) avec le même niveau d'exigence que le second facteur principal, et les cookies "se souvenir de cet appareil" avec une valeur imprévisible, signée et liée au compte et à l'appareil.
- S'assurer que tous les flux sensibles du compte (réinitialisation de mot de passe, changement d'email, accès API) imposent la même exigence de second facteur, sans chemin parallèle qui la contourne.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-01
- [x] Practitioner ✅ 2026-10-01
- [ ] Expert

## Journal des labs
- 

## Mes notes
- 

## Liens
- [[Authentication]]
- [[Access-control]]
- [[Autres vulnérabilité dans les mécanismes d'authentification]]
