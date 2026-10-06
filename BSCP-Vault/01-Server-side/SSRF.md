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


## Comment détecter
- 

## Comment exploiter (principe)
- **Attaque contre le serveur lui-même**
	- L'attaquant amène l'application à envoyer une requête HTTP vers le serveur qui l'héberge via son interface réseau de bouclage.
	- L'URL fournie contient généralement `127.0.0.1` ou `localhost`
- **Exemple: Vérification de stock dans une boutique en ligne**
	- Pour afficher ses stocks, l'application interroge des API REST.
	- Le navigateur envoie une requête `POST /product/stock` dont le paramètre `stockApi` contient l'URL du point de terminaison.
	- Le serveur envoie la requête à cette URL et récupère l'état des stocks et les renvoie à l'utilisateur.
- **L'attaque**
	- L'attaquant modifie le paramètre pour y mettre une URL locale au serveur: `stockApi=http://localhost/admin`
	- Ainsi Le serveur récupère le contenu de `/admin` et le renvoie à l'attaquant.
- **Pourquoi cela marche ?**
	- Normalement, `/admin` est accessible uniquement aux utilisateur qui se sont authentifiés.
	- Quand la requête provient de la machine locale, les contrôles d'accès habituels sont contournés.
- **Pourquoi les applications font confiance à la machine locale**
	- Le contrôle d'accès peut être implémenté dans un composant distinct, situé en amont du serveur d'applications. Une connexion établie directement depuis le serveur contourne ce contrôle.
	- l'application peut autoriser un accès administratif sans authentification à tout utilisateur provenant de la machine locale. Un administrateur peut ainsi restaurer le système s'il perd ses identifiants
	- L'interface d'administration peut écouter sur un autre port que l'application principale et n'être pas accessible directement aux utilisateurs.
	- Ces relations de confiance, où les requêtes locales sont traités différement des requêtes ordinaires, font souvent de la SSRF une faille cririque.
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
