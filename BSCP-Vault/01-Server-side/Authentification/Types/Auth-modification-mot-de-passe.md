---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification - modification du mot de passe

## En bref
- Le processus classique demande le mot de passe actuel, puis le nouveau mot de passe saisi deux fois.
- Il repose sur le même mécanisme de vérification que la page de connexion, donc sur les mêmes failles.
- Note parente : [[Autres-mecanismes-authentification]]

## Comment détecter
- Vérifier si le nom d'utilisateur est transmis dans un champ masqué.
- Comparer les messages d'erreur selon que le mot de passe actuel est correct ou non.

## Comment exploiter (principe)
- Le processus classique demande le mot de passe actuel, puis le nouveau mot de passe saisi deux fois.
- Ces pages reposent sur le même mécanisme de vérification qu'une page de connexion classique, et sont donc exposées aux mêmes techniques d'attaque (énumération, brute-force, etc.).
- Cas typique de faille : le nom d'utilisateur est transmis dans un champ masqué du formulaire. Un attaquant peut modifier cette valeur dans la requête pour cibler un utilisateur arbitraire, sans être connecté à sa victime.
- **Oracle par message d'erreur** : si le mot de passe actuel est faux, le message est `Current password is incorrect` ; s'il est correct mais que les deux nouveaux mots de passe diffèrent, le message est `New passwords do not match`. Cette différence permet de forcer le mot de passe actuel avec Intruder.

## Pièges et points d'attention BSCP
- Garder les deux nouveaux mots de passe différents entre eux : c'est cette différence qui évite le verrouillage et produit le message exploitable.
- Vérifier le champ masqué du nom d'utilisateur : simple à manquer.

## Labs PortSwigger
- [[Autres-mecanismes-authentification#Lab 2 - Force brute du mot de passe via la modification du mot de passe]] (Practitioner)

## Liens
- [[Autres-mecanismes-authentification]]
- [[Auth-reinitialisation-mot-de-passe]]
