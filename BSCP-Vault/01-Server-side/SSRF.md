---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: à faire
---
# Server-side request forgery (SSRF)

## En bref
- SSRF est une faille qui permet à un attaquant d'amener une application côté serveur et à envoyer des requêtes vers une destination non prévue.
- Le serveur est amené à se connecter à des services à usage interne, au sein de l'infrastructure.
- Le serveur peut être forcé à se connecter à des systèmes externes.
- L'impact est qu'une fuite de données sensible est possible.
![](SSRF.png)


## Types et variantes
- **Attaque contre le serveur lui-même**
	- L'attaquant amène l'application à envoyer une requête HTTP vers le serveur qui l'héberge via son interface réseau de bouclage.
	- L'URL fournie contient généralement `127.0.0.1` ou `localhost`
- **Exemple: Vérification de stock dans une boutique en ligne**
	- Pour afficher ses stocks, l'application interroge des API REST.
	- Le navigateur envoie une requête `POST /product/stock` do

## Comment détecter
- 

## Comment exploiter (principe)
- 

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- 

## Liens
- [[Command-injection]]
- [[XXE-injection]]
- [[Web-cache-poisoning]]
