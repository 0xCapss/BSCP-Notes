---
tags: [bscp, server-side, authentication, mfa]
niveau: expert
statut: en cours
---
# Multi-factor authentication

## En bref
- Méthode de sécurité qui demande à un utilisateur de fournir plusieurs preuves de son identité, en combinant des facteurs de catégories différentes (connaissance, possession, inhérent), pour accéder à un compte ou une application.
- À distinguer de la "two-step verification" (deux étapes mais avec des facteurs de la même catégorie, par exemple un mot de passe puis une réponse à une question secrète) : ce n'est pas la même robustesse, même si les deux ajoutent une étape de connexion.
- Un défaut d'implémentation du second facteur peut annuler tout l'intérêt de la mesure et ramener la sécurité au niveau d'une authentification par mot de passe seul.
- Formes de code et espace de recherche :
	- TOTP (Time-based One-Time Password) : code généré localement par une application à partir d'un secret partagé et de l'heure ; typiquement 6 chiffres renouvelés toutes les 30 secondes, avec une tolérance d'une ou deux fenêtres côté serveur. Espace de 1 000 000 de valeurs, mais fenêtre de validité courte.
	- Code envoyé par SMS ou par e-mail : généré par le serveur et transmis ; longueur et durée de vie fixées par l'application (4 chiffres dans le lab 11 de [[Authentication]], soit 10 000 valeurs).
	- Notification push : pas de code à deviner, la faiblesse se situe dans la validation par l'utilisateur.
	- La longueur du code et sa durée de vie déterminent le nombre d'essais nécessaires et la fenêtre dont dispose un brute force.
- Inscription et gestion du second facteur : qui peut l'activer, le désactiver ou le réinitialiser, et avec quelle preuve (mot de passe actuel, code existant) ? Un chemin de réinitialisation du second facteur plus faible que le facteur lui-même annule la protection.

## Types et variantes
- Facteur de possession matérialisé par :
	- Une application d'authentification générant un OTP (One-Time Password), par exemple Microsoft/Google Authenticator.
	- Un token matériel dédié (type Yubikey).
	- Un code reçu par SMS, vulnérable au SIM swapping (prise de contrôle de la carte SIM de la victime) et à l'interception par phishing en temps réel (proxy transparent type Evilginx qui relaie le code à la volée).
	- Une notification push à valider sur un appareil de confiance.
- Mécanismes de secours (backup/recovery codes) : généralement affichés une seule fois à la configuration, parfois réutilisables ou non invalidés après usage, souvent moins protégés que l'OTP principal (pas de rate limiting sur leur saisie, format prévisible).
- Cookie "se souvenir de cet appareil" (trusted device) : permet de sauter la demande de second facteur sur les connexions suivantes depuis le même navigateur ; sa sécurité dépend entièrement de la robustesse de la valeur (imprévisible, signée, liée au compte et à l'appareil).

## Comment détecter
- Vérifier si des pages ou fonctions censées être accessibles uniquement après validation complète (étape 1 + étape 2) répondent normalement en y naviguant directement juste après l'étape 1, sans jamais soumettre le second facteur.
- Vérifier si l'identité du compte visé par la deuxième étape est portée par une valeur modifiable côté client (cookie, paramètre caché, champ de formulaire) plutôt que dérivée d'un état de session vérifié côté serveur.
- Déterminer l'espace de recherche du code de vérification (nombre de chiffres) et tester s'il existe un rate limiting : sans lui, le code devient vulnérable au brute force. Mesurer ce rate limiting :
	- compter le nombre d'essais erronés avant coupure (message, déconnexion, account lockout) ;
	- déterminer sa portée : par session, par compte ou par IP ;
	- vérifier si une nouvelle connexion (nouvelle session) remet le compteur à zéro.
- Mesurer l'expiration : demander un code, attendre une durée connue (par exemple plusieurs minutes), puis le soumettre ; noter la durée à partir de laquelle il est refusé.
- Vérifier si un code de vérification reste valide après un usage réussi (pas d'invalidation immédiate), si un code d'une session fonctionne dans une autre session, et si l'ancien code est invalidé quand un nouveau code est demandé.
- Si la vérification semble se faire côté client (JS) ou renvoie un indicateur de statut explicite, intercepter la réponse pour voir si un champ comme `"verified":false` est manipulable avant d'être interprété par l'application.
- Tester si un cookie "se souvenir de cet appareil" existe et s'il est prévisible, rejouable sur un autre navigateur, ou non lié de façon vérifiable au compte.
- Vérifier que les autres flux sensibles du compte (réinitialisation de mot de passe, changement d'email, API mobile parallèle) imposent eux aussi la validation du second facteur, et n'offrent pas un chemin parallèle permettant un bypass du second facteur.

## Comment exploiter (principe)
1. Après l'étape 1, tenter d'atteindre directement une page protégée.
2. Chercher une valeur modifiable qui désigne le compte entre les deux étapes.
3. Évaluer l'espace de recherche du code et l'existence d'un rate limiting.
4. Tester la réponse de vérification, les codes de secours et le cookie « se souvenir de cet appareil ».

| Type | Idée clé | Signal attendu | Limite | Note | Niveau |
| --- | --- | --- | --- | --- | --- |
| Accès direct | forcer l'URL post-connexion | page de compte affichée sans code | corrigé si le serveur revérifie la deuxième étape | [[MFA-acces-direct]] | Apprentice |
| Logique défaillante entre étapes | `Cookie: verify=carlos` | `302` après le bon code, avec l'identité modifiée | rate limiting sur le code, durée de vie du code | [[MFA-logique-defaillante]] | Practitioner |
| Brute force du code | `0000` à `9999`, macro | `302` sur le bon code | rate limiting par session, expiration, nombre de requêtes à envoyer, plusieurs passes possibles | [[MFA-brute-force-code]] | Expert |
| Réponse manipulable et secours | `"verified":true`, backup codes | accès accordé sans code valide | rare ; dépend de l'implémentation | [[MFA-reponse-et-secours]] | Practitioner |
| Flux parallèles | réinitialisation du mot de passe, OAuth, API mobile | session ouverte sans code | dépend de l'application | voir [[Access-control]], [[OAuth]] | non classé |

## Pièges et points d'attention BSCP
- Toujours distinguer les deux causes qui peuvent se combiner dans un même scénario : l'absence de rate limiting sur le code, et la logique défaillante de liaison d'identité entre les deux étapes. Ce sont deux vulnérabilités séparées, même quand l'exploitation les enchaîne.
- L'absence de rate limiting et l'absence d'expiration du code sont deux contrôles différents : un code sans rate limiting mais qui expire vite reste praticable, un code sans expiration mais protégé par un rate limiting beaucoup moins.
- Ne pas se limiter au cookie comme seul support possible de la faille de liaison entre étapes : tester aussi les paramètres cachés et l'état de session côté serveur.
- Un brute force du code se heurte au rate limiting par session et à l'expiration. Les macros et la gestion de session de Burp sont alors le moyen de rétablir une session valide entre les essais. Le coût est élevé : 10 000 requêtes pour un code à 4 chiffres, à prévoir dans la gestion du temps à l'examen.

## Prévention
Chaque mesure est rattachée à la variante qu'elle neutralise.
- Accès direct : ne jamais considérer la session comme authentifiée tant que toutes les étapes requises n'ont pas été validées côté serveur ; refuser l'accès à toute ressource protégée tant que le second facteur n'est pas confirmé.
- Logique défaillante entre étapes : dériver l'identité du compte en cours de vérification depuis un état de session signé et vérifié côté serveur, jamais depuis une valeur modifiable par le client (cookie, paramètre, champ caché).
- Brute force du code : appliquer un rate limiting strict sur le code de vérification et sur les codes de secours, par compte et non par seule session, avec account lockout ou délai croissant après quelques échecs. Faire expirer les codes rapidement et les invalider après un usage réussi ou quand un nouveau code est émis.
- Réponse manipulable : ne jamais faire reposer la décision finale de validation sur une donnée renvoyée au client ; la vérification et la décision d'accès doivent être strictement côté serveur.
- Secours et appareil de confiance : protéger les codes de secours avec le même niveau d'exigence que le second facteur principal, et les cookies "se souvenir de cet appareil" avec une valeur imprévisible, signée et liée au compte et à l'appareil.
- Flux parallèles : s'assurer que tous les flux sensibles du compte (réinitialisation de mot de passe, changement d'email, accès API) imposent la même exigence de second facteur, sans chemin parallèle permettant un bypass.
- Préférer une application qui génère le code (TOTP) au SMS, vulnérable au SIM swapping et au phishing en temps réel.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-01
- [x] Practitioner ✅ 2026-10-01
- [ ] Expert

## Journal des labs
- Les trois labs de 2FA (contournement simple, logique défaillante, force brute) sont décrits dans le journal de [[Authentication]] (labs 10 à 12).

## Liens
- [[Authentication]]
- [[Access-control]] - accès direct à une page protégée sans passer par la deuxième étape
- [[OAuth]] - connexion déléguée qui peut ne pas exiger le second facteur
- [[Autres-mecanismes-authentification]]
- [[MFA-acces-direct]]
- [[MFA-logique-defaillante]]
- [[MFA-brute-force-code]]
- [[MFA-reponse-et-secours]]
