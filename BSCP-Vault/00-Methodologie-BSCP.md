---
tags: [bscp, méthode, examen]
---
# Méthodologie BSCP

> Vérifie le format officiel à jour (durée, nombre d'applications, seuil de
> réussite) sur le site de PortSwigger avant de réserver. Ce qui suit est la
> méthode de fond, stable, pas le règlement exact.

## Nature de l'examen
Examen pratique : plusieurs applications web à compromettre en un temps limité,
avec Burp Suite. Chaque application se compromet par étapes, typiquement :
1. Obtenir l'accès à un compte utilisateur de bas niveau.
2. Élever vers un compte administrateur.
3. Atteindre l'objectif final (souvent lecture d'un fichier ou exécution).

Chaque étape se résout en chaînant les vulnérabilités vues dans la Web Security
Academy. La clé n'est pas de connaître une faille isolée, mais de savoir la
repérer vite et l'enchaîner.

## Déroulé recommandé le jour J
1. Cartographier l'application (sitemap, fonctionnalités, rôles, entrées).
2. Passer chaque surface en revue avec une checklist de vulnérabilités.
3. Prioriser les pistes les plus probables selon les indices observés.
4. Exploiter, confirmer, puis avancer d'une étape. Ne pas s'acharner : si une
   piste bloque au-delà d'un temps fixé, en changer et y revenir.
5. Garder Repeater organisé et noter chaque accès obtenu.

## Gestion du temps
- Fixe-toi un budget par application et par étape, et tiens-le.
- Une piste qui ne donne rien après un temps fixé se met de côté, pas en boucle.
- Réussir toutes les applications prime sur l'élégance : vise l'objectif.

## Checklist par surface
- Point d'entrée utilisateur : SQLi, XSS, injection, traversal, upload ?
- Authentification et session : logique, reset, JWT, OAuth ?
- Contrôle d'accès : IDOR, fonctions privilégiées accessibles ?
- Logique métier : hypothèses contournables, race conditions ?
- Entrées portant des URL : SSRF, host header ?
- Rendu de données : contexte d'injection, DOM ?

## Avant l'examen
- Avoir fait un maximum de labs, surtout Practitioner et quelques Expert.
- Refaire à blanc une compromission complète en conditions de temps.
- Relire les notes au statut `en cours` pour les passer à `maîtrisé`.
