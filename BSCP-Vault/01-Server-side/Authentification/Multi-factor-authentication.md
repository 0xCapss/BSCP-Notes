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
1. Après l'étape 1, tenter d'atteindre directement une page protégée.
2. Chercher une valeur modifiable qui désigne le compte entre les deux étapes.
3. Évaluer l'espace de recherche du code et l'existence d'une limitation de tentatives.
4. Tester la réponse de vérification, les codes de secours et le cookie « se souvenir de cet appareil ».

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Accès direct | forcer l'URL post-connexion | [[MFA-acces-direct]] | Apprentice |
| Logique défaillante entre étapes | `Cookie: verify=carlos` | [[MFA-logique-defaillante]] | Practitioner |
| Brute force du code | `0000` à `9999`, macro | [[MFA-brute-force-code]] | Expert |
| Réponse manipulable et secours | `"verified":true`, backup codes | [[MFA-reponse-et-secours]] | Practitioner |

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
- Les trois labs de 2FA (contournement simple, logique défaillante, force brute) sont décrits dans le journal de [[Authentication]] (labs 10 à 12).

## Mes notes
- 

## Liens
- [[Authentication]]
- [[Access-control]]
- [[Autres-mecanismes-authentification]]
- [[MFA-acces-direct]]
- [[MFA-logique-defaillante]]
- [[MFA-brute-force-code]]
- [[MFA-reponse-et-secours]]
