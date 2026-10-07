---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: à faire
---
# Surfaces d'attaque cachées pour la SSRF

## En bref
- De nombreuses SSRF sont faciles à détecter : le trafic normal de l'application contient des paramètres de requête avec des URL complètes.
- D'autres sont plus difficiles à repérer car l'URL n'apparaît pas telle quelle dans la requête.
- Note parente : [[SSRF]]

## URL partielles
- Une application peut n'intégrer dans les paramètres de requête qu'un nom d'hôte ou une partie d'un chemin d'URL.
- Côté serveur, cette valeur est insérée dans une URL complète qui fait ensuite l'objet de la requête.
- Surface d'attaque : si la valeur est facilement identifiable comme un nom d'hôte ou un chemin d'URL, la surface est évidente.
- Limite : l'attaquant ne contrôle pas toute l'URL, ce qui réduit souvent l'exploitation (le chemin ou le schéma restent imposés).

## URL dans les formats de données
- Certaines applications transmettent des données dans des formats dont la spécification autorise l'inclusion d'URL, que l'analyseur du format peut ensuite solliciter.
- Exemple XML :
	- Format largement utilisé dans les applications web pour transmettre des données structurées du client au serveur.
	- Une application qui accepte et analyse du XML peut être vulnérable à une injection XXE.
	- Elle peut aussi être vulnérable à une SSRF via XXE (voir [[XXE-injection]]).

## SSRF via l'en-tête Referer
- Certaines applications utilisent des logiciels d'analyse côté serveur pour suivre les visiteurs.
- Ces logiciels enregistrent souvent l'en-tête `Referer` des requêtes, afin de suivre les liens entrants.
- Surface d'attaque :
	- Ils accèdent fréquemment aux URL tierces présentes dans l'en-tête `Referer`.
	- Le but est généralement d'analyser le contenu des sites référents, y compris le texte d'ancrage des liens entrants.
	- L'en-tête `Referer` est donc souvent une surface utile pour la SSRF, généralement de type aveugle, à tester avec Burp Collaborator (voir [[SSRF-aveugle]]).

## Pièges et points d'attention BSCP
- Penser aussi aux autres en-têtes que le serveur peut résoudre : `Host`, `X-Forwarded-Host` (voir [[HTTP-Host-header]]).
- Penser aux fonctionnalités qui « importent » ou « prévisualisent » une adresse : import par URL, webhooks, génération de PDF, aperçu de lien, avatar distant.

## Labs PortSwigger
- [[SSRF#Lab 5 - SSRF aveugle avec détection hors bande]] (Practitioner)

## Liens
- [[SSRF]]
- [[SSRF-aveugle]]
- [[XXE-injection]]
- [[HTTP-Host-header]]
