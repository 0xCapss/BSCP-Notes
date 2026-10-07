---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification - maintien de la connexion (« Remember me »)

## En bref
- Fonctionnalité qui permet à un utilisateur de rester connecté après avoir fermé son navigateur, via un token stocké dans un cookie persistant.
- Note parente : [[Autres-mecanismes-authentification]]

## Comment détecter
- Se connecter avec la case cochée et examiner le cookie obtenu.
- Décoder le cookie (base64, hexadécimal) et comparer avec le username, le mot de passe, un horodatage.
- Créer plusieurs comptes ou se connecter plusieurs fois pour repérer la structure.

## Comment exploiter (principe)
- Fonctionnalité qui permet à un utilisateur de rester connecté après avoir fermé son navigateur, via un token stocké dans un cookie persistant.
- Ce cookie peut être généré par le site à partir de valeurs statiques (par exemple le nom d'utilisateur suivi d'un horodatage) : un attaquant qui observe plusieurs cookies peut en déduire la structure de génération.
- Le cookie peut être chiffré, mais un encodage réversible comme le base64 n'offre aucune protection réelle.
- Si le mot de passe est haché sans salt, des listes de mots de passe courants (rainbow tables) permettent de le retrouver. D'où l'importance du salt.
- Via une faille XSS, un attaquant peut dérober le cookie "Se souvenir de moi" d'un autre utilisateur et en étudier la structure pour le forger.
- **Forme typique** : `base64(username:md5(mot de passe))`. Un attaquant qui connaît le username peut forger le cookie de chaque mot de passe candidat et le tester sans passer par le formulaire de connexion (donc sans verrouillage de compte).
- **Craquage hors ligne** : le cookie de la victime volé (via XSS stocké), une fois décodé, contient le hash du mot de passe, que l'on casse hors ligne avec une base de hashs ou un outil de craquage.

## Pièges et points d'attention BSCP
- Dans Intruder, appliquer des règles de traitement de payload dans l'ordre : hachage MD5, ajout du préfixe `username:`, encodage base64.
- Repérer le succès par la présence d'un texte propre à la page connectée (par exemple `Update email`).

## Labs PortSwigger
- [[Authentication#Lab 7 - Force brute d'un cookie de connexion persistante]] (Practitioner)
- [[Authentication#Lab 8 - Craquage de mot de passe hors ligne]] (Practitioner)

## Liens
- [[Autres-mecanismes-authentification]]
- [[XSS]]
- [[Auth-reinitialisation-mot-de-passe]]
