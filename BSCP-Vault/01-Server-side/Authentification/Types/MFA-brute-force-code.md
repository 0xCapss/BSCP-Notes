---
tags: [bscp, server-side, authentication]
niveau: expert
statut: à faire
---
# Authentification multifacteur - brute force du code de vérification

## En bref
- Un code de vérification a souvent 4 ou 6 chiffres : espace de recherche de 10 000 à 1 000 000.
- Note parente : [[Multi-factor-authentication]]

## Comment détecter
- Déterminer le nombre de chiffres et tester s'il existe une limitation de tentatives (déconnexion après plusieurs échecs, verrouillage).

## Comment exploiter (principe)
- Un code de vérification est souvent un nombre à 4 ou 6 chiffres, soit un espace de recherche de 10 000 à 1 000 000 de valeurs : praticable avec Burp Intruder (attaque Sniper, position sur le code, type de payload Numbers) si rien d'autre ne limite les tentatives.
- Condition indispensable à cette attaque : l'absence de limitation de tentatives sur la saisie du code. Sans cette faiblesse, le brute force est impraticable (voir [[Authentication]] pour la logique de rate limiting déjà vue sur les mots de passe).
- Burp Intruder standard peut être trop lent pour couvrir tout l'espace en conditions d'examen ; Turbo Intruder est souvent nécessaire pour une attaque praticable dans le temps imparti.
- Vérifier aussi si un code reste valide après un usage réussi (pas d'invalidation) ou au-delà d'une fenêtre de temps large (absence d'expiration) : deux faiblesses distinctes de la limitation de tentatives, qui élargissent la fenêtre d'exploitation.

## Pièges et points d'attention BSCP
- Si la session est invalidée après quelques échecs, utiliser une macro Burp et une règle de gestion de session pour se reconnecter avant chaque tentative, avec une seule requête simultanée.
- L'absence de limitation de tentatives et l'absence d'expiration du code sont deux contrôles différents.

## Labs PortSwigger
- [[Authentication#Lab 12 - Contournement de la 2FA par force brute]] (Expert)

## Liens
- [[Multi-factor-authentication]]
- [[MFA-logique-defaillante]]
