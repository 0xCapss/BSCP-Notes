---
tags: [bscp, server-side, authentication]
niveau: apprentice
statut: à faire
---
# Authentification multifacteur - accès direct après la première étape

## En bref
- L'application considère l'utilisateur comme « quasi connecté » dès la validation du mot de passe, avant la validation du second facteur.
- Note parente : [[Multi-factor-authentication]]

## Comment détecter
- Se connecter avec ses propres identifiants, noter l'URL de la page post-connexion (`/my-account`), puis tester d'y aller directement avant de saisir le code.

## Comment exploiter (principe)
- Si l'application considère l'utilisateur comme "quasi connecté" dès la validation du mot de passe, avant la validation du second facteur, il est parfois possible d'accéder directement aux pages ou fonctions réservées aux utilisateurs pleinement authentifiés en forçant la navigation vers leur URL, sans jamais soumettre de code de vérification.

## Pièges et points d'attention BSCP
- Utiliser un compte de la victime dont on connaît le mot de passe mais pas le second facteur, puis naviguer directement vers la page protégée.

## Labs PortSwigger
- [[Authentication#Lab 10 - Contournement simple de la 2FA]] (Apprentice)

## Liens
- [[Multi-factor-authentication]]
- [[MFA-logique-defaillante]]
