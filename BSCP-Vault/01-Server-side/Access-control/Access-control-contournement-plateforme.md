---
tags: [bscp, server-side, access-control]
niveau: practitioner
statut: à faire
---
# Contrôle d'accès - contournement lié à la plateforme (URL et méthode HTTP)

## En bref
- Certaines applications appliquent le contrôle d'accès au niveau de la plateforme (pare-feu applicatif, proxy, framework) en restreignant l'accès à des URL et méthodes HTTP précises. Ces règles sont contournables.
- Note parente : [[Access-control]]

## Comment détecter
- Comparer les réponses d'une URL sensible avec différents en-têtes et méthodes.
- Envoyer une requête sur `/` avec `X-Original-URL: /invalid` : si la réponse est un « not found » de l'application (et non du front-end), l'en-tête est pris en compte.

## Comment exploiter (principe)
- **Contrôle d'accès basé sur l'URL** : certains frameworks prennent en charge des en-têtes non standard qui réécrivent l'URL, comme `X-Original-URL` ou `X-Rewrite-URL`. Si le front-end bloque `/admin` mais que le back-end suit l'en-tête, une requête vers `/` avec `X-Original-URL: /admin` contourne le blocage.
- **Contrôle d'accès basé sur la méthode HTTP** : la règle bloque `POST` mais pas `GET`, ou l'inverse. Changer la méthode (clic droit, « Change request method » dans Burp) peut contourner le contrôle.
- **Variantes de chemin** : casse (`/ADMIN/DELETEUSER`), barre finale (`/admin/deleteUser/`), suffixe (`/admin/deleteUser.anything`).

## Pièges et points d'attention BSCP
- Avec `X-Original-URL`, les paramètres de requête restent dans la ligne de requête réelle : `GET /?username=carlos` avec `X-Original-URL: /admin/delete`.
- Tester toutes les méthodes (`GET`, `POST`, `PUT`, `DELETE`) sur chaque endpoint sensible.

## Labs PortSwigger
- [[Access-control#Lab 10 - Contrôle d'accès par URL contournable]] (Practitioner)
- [[Access-control#Lab 11 - Contrôle d'accès par méthode contournable]] (Practitioner)

## Liens
- [[Access-control]]
- [[HTTP-Host-header]]
- [[HTTP-request-smuggling]]
