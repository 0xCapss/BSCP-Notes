---
tags: [bscp, server-side, access-control]
niveau: practitioner
statut: à faire
---
# Contrôle d'accès - processus en plusieurs étapes et en-tête Referer

## En bref
- Certains sites réalisent une opération sensible en plusieurs étapes (saisie, confirmation) et ne contrôlent l'accès que sur certaines d'entre elles.
- D'autres se fient à l'en-tête `Referer` pour décider si une requête est légitime.
- Note parente : [[Access-control]]

## Comment détecter
- Décomposer chaque action sensible en ses étapes avec Burp, puis rejouer chaque étape séparément avec une session de faible privilège.
- Retirer ou modifier l'en-tête `Referer` d'une requête sensible et observer si le résultat change.

## Comment exploiter (principe)
- **Processus en plusieurs étapes** : le contrôle est appliqué sur les premières étapes mais pas sur la dernière (par exemple la confirmation). L'attaquant saute les étapes protégées et envoie directement la requête de la dernière étape avec sa propre session.
- **Contrôle basé sur le Referer** : l'application vérifie que `Referer` contient une URL d'administration. Cet en-tête est contrôlé par l'attaquant : il suffit de le fournir (`Referer: https://site/admin`) dans la requête.

## Pièges et points d'attention BSCP
- Bien rejouer la requête de confirmation avec le cookie de session du compte à faible privilège, pas celui de l'administrateur.
- Ne jamais considérer `Referer` comme une preuve de provenance : c'est un en-tête client.

## Labs PortSwigger
- [[Access-control#Lab 12 - Processus en plusieurs étapes sans contrôle d'accès sur une étape]] (Practitioner)
- [[Access-control#Lab 13 - Contrôle d'accès basé sur l'en-tête Referer]] (Practitioner)

## Liens
- [[Access-control]]
- [[Business-logic]]
- [[CSRF]]
