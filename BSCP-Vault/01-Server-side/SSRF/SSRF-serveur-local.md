---
tags: [bscp, server-side, ssrf]
niveau: apprentice
statut: à faire
---
# SSRF contre le serveur local

## En bref
- L'attaquant amène l'application à envoyer une requête HTTP vers le serveur qui l'héberge, via son interface réseau de bouclage (loopback).
- L'URL fournie contient généralement `127.0.0.1` ou `localhost`.
- Note parente : [[SSRF]]

## Comment détecter
- Repérer une fonctionnalité qui reçoit une URL complète en paramètre (exemple : `stockApi` dans une vérification de stock).
- Remplacer l'URL par `http://localhost/` et comparer la réponse avec l'URL d'origine.
- Tester ensuite des chemins connus mais inaccessibles de l'extérieur : `/admin`, `/console`, `/actuator`.

## Comment exploiter (principe)
- **Exemple : vérification de stock dans une boutique en ligne**
	- Pour afficher ses stocks, l'application interroge des API REST.
	- Le navigateur envoie une requête `POST /product/stock` dont le paramètre `stockApi` contient l'URL du point de terminaison.
	- Le serveur envoie la requête à cette URL, récupère l'état des stocks et le renvoie à l'utilisateur.
- **L'attaque**
	- L'attaquant remplace le paramètre par une URL locale au serveur : `stockApi=http://localhost/admin`.
	- Le serveur récupère le contenu de `/admin` et le renvoie à l'attaquant.
- **Pourquoi cela marche ?**
	- Normalement, `/admin` n'est accessible qu'aux utilisateurs authentifiés.
	- Quand la requête provient de la machine locale, les contrôles d'accès habituels sont contournés.
- **Pourquoi les applications font confiance à la machine locale**
	- Le contrôle d'accès peut être implémenté dans un composant distinct, situé en amont du serveur d'applications. Une connexion établie directement depuis le serveur contourne ce contrôle.
	- L'application peut autoriser un accès administratif sans authentification à tout utilisateur provenant de la machine locale. Un administrateur peut ainsi restaurer le système s'il perd ses identifiants.
	- L'interface d'administration peut écouter sur un autre port que l'application principale et ne pas être accessible directement aux utilisateurs.
	- Ces relations de confiance, où les requêtes locales sont traitées différemment des requêtes ordinaires, font souvent de la SSRF une faille critique.

## Pièges et points d'attention BSCP
- Si `localhost` ou `127.0.0.1` sont bloqués, passer à [[SSRF-filtre-liste-noire]].
- Si l'URL doit appartenir à un domaine précis, passer à [[SSRF-filtre-liste-blanche]] ou [[SSRF-redirection-ouverte]].
- Lire le HTML de l'interface d'administration pour trouver l'URL exacte de l'action à déclencher (suppression d'utilisateur par exemple).

## Labs PortSwigger
- [[SSRF#Lab 1 - SSRF basique contre le serveur local]] (Apprentice)

## Liens
- [[SSRF]]
- [[SSRF-systemes-back-end]]
- [[Access-control]]
