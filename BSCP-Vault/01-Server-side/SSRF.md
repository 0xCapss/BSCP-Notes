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
## Impact
- Une attaque SSRF réussie peut entraîner des actions non autorisée aux données au sein d'une organisation.
- Cela peut se produire dans l'application elle même, ou sur d'autres système back-end avec lesquels elle communique.
- Dans certains cas, la SSRF peut permettre à un attaquant d'exécuter des commandes arbitraires.
- Ces attaques peuvent sembler provenir de l'organisation qui héberge l'application vulnérable (l'adresse IP source est celle du serveur).


## Types et variantes
### Attaque SSRF courantes
- Les attaques SSRF exploitent souvent des relations de confiance pour étendre l'attaque à partir de l'application vulnérable.
- Elles permettent d'effectuer des actions non autorisées.
- Les relations de confiance sont exploitées : 
	- A travers le serveur lui-

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
	- Ces relations de confiance, où les requêtes locales sont traités different des requêtes ordinaires, font souvent de la SSRF une faille cririque.
	
- **SSRF contre d'autres systèmes back-end**:
	- Le serveur d'application peut interagir avec des systèmes back-end qui ne sont pas directement accessibles aux utilisateurs.
	- Ces système ont souvent des adresses IP privées non routables.
- **Pourquoi est-ce vulnérable?**
	- Ils sont généralement protégés par la topologie du réseau, d'où un niveau de sécurité souvent plus faible.
	- Beaucoup contiennent des fonctionnalités sensibles accessibles sans authentification à quiconque peut interagir avec eux.
- **Exemple**
	- Une interface d'administration existe à l'URL back-end `http://192.168.0.68/admin`.
	- L'attaquant envoie une requête `POST /product/stock` avec `stockApi=http://192.168.0.68/admin`.
	- Le serveur d'applications relaie la requête vers ce système interne, ce qui donne accès à l'interface d'administration.
## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Basic SSRF against the local server
This lab has a stock check feature which fetches data from an internal system.
To solve the lab, change the stock check URL to access the admin interface at `http://localhost/admin` and delete the user `carlos`.

1. Browse to `/admin` and observe that you can't directly access the admin page.
2. Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
3. Change the URL in the `stockApi` parameter to `http://localhost/admin`. This should display the administration interface.
4. Read the HTML to identify the URL to delete the target user, which is:
    
    `http://localhost/admin/delete?username=carlos`
5. Submit this URL in the `stockApi` parameter, to deliver the SSRF attack.
### Lab: Basic SSRF against another back-end system
This lab has a stock check feature which fetches data from an internal system.
To solve the lab, use the stock check functionality to scan the internal `192.168.0.X` range for an admin interface on port `8080`, then use it to delete the user `carlos`.

1. Visit a product, click **Check stock**, intercept the request in Burp Suite, and send it to Burp Intruder.
2. Change the `stockApi` parameter to `http://192.168.0.1:8080/admin` then highlight the final octet of the IP address (the number `1`) and click **Add §**.
3. In the **Payloads** side panel, change the payload type to **Numbers**, and enter 1, 255, and 1 in the **From** and **To** and **Step** boxes respectively.
4. Click  **Start attack**.
5. Click on the **Status** column to sort it by status code ascending. You should see a single entry with a status of `200`, showing an admin interface.
6. Click on this request, send it to Burp Repeater, and change the path in the `stockApi` to: `/admin/delete?username=carlos`


## Liens
- [[Command-injection]]
- [[XXE-injection]]
- [[Web-cache-poisoning]]
