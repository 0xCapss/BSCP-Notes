---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification multifacteur - réponse manipulable et mécanismes de secours

## En bref
- La décision de validation repose sur une valeur renvoyée au client, ou un mécanisme de secours est plus faible que le second facteur principal.
- Note parente : [[Multi-factor-authentication]]

## Comment détecter
- Intercepter la réponse de vérification et repérer un indicateur de statut (par exemple `"verified":false`).
- Tester les codes de secours, le cookie « se souvenir de cet appareil » et les autres flux du compte.

## Comment exploiter (principe)
- Quand la décision "code valide ou non" repose sur une valeur renvoyée au client plutôt que strictement vérifiée et appliquée côté serveur, intercepter la réponse et modifier l'indicateur de statut (par exemple un corps JSON `"verified":false` changé en `true`) peut suffire à obtenir l'accès sans connaître le bon code. Faille de confiance dans la réponse, à bien distinguer de la logique défaillante entre étapes vue plus haut.

### Contournement via les mécanismes de secours ou de confiance
- Les codes de secours (backup codes) contournent entièrement le second facteur principal ; s'ils ne sont pas soumis aux mêmes protections (limitation de tentatives, format suffisamment long), ils deviennent le maillon faible à cibler plutôt que l'OTP principal.
- Un cookie "se souvenir de cet appareil" prévisible, non signé, ou rejouable sur un autre navigateur permet de sauter complètement la demande de second facteur sur les connexions suivantes.
- Vérifier si d'autres flux du compte (réinitialisation de mot de passe, changement d'email, API mobile) imposent eux aussi le second facteur : un chemin parallèle qui ne le fait pas revient à contourner toute la protection (voir [[Autres-mecanismes-authentification]]).

## Pièges et points d'attention BSCP
- Faille de confiance dans la réponse à bien distinguer de la logique défaillante entre étapes.
- Un faux « MFA » combinant deux facteurs de la même catégorie n'est qu'une vérification en deux étapes.

## Labs PortSwigger
- Aucun lab dédié dans la formation : concept à maîtriser, notamment pour l'examen.

## Liens
- [[Multi-factor-authentication]]
- [[MFA-acces-direct]]
