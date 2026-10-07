---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification multifacteur - logique défaillante entre les deux étapes

## En bref
- Le site ne vérifie pas que c'est bien le même utilisateur qui effectue les deux étapes.
- Note parente : [[Multi-factor-authentication]]

## Comment détecter
- Repérer, entre l'étape 1 et l'étape 2, une valeur modifiable qui désigne le compte (cookie, paramètre, champ masqué).

## Comment exploiter (principe)
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

## Pièges et points d'attention BSCP
- Ce sont deux vulnérabilités distinctes (liaison d'identité et absence de limitation des tentatives) même quand l'exploitation les enchaîne.
- Ne pas se limiter au cookie : tester aussi les paramètres cachés et l'état de session.

## Labs PortSwigger
- [[Authentication#Lab 11 - Logique défaillante de la 2FA]] (Practitioner)

## Liens
- [[Multi-factor-authentication]]
- [[MFA-brute-force-code]]
