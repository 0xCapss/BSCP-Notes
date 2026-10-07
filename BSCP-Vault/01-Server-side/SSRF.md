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
	- A travers le serveur lui-même
	- Celles qui concernent d'autres systèmes back-end au sein de la même organisation.
- Il est fréquent de rencontrer des applications présentant un comportement SSRF et intégrant des mesures de protection destinées à empêcher toute exploitation malveillante. Souvent, ces mesures de protection peuvent être contournées.
### SSRF avec des filtres d'entrée basés sur une liste noire
- Certaines applications bloquent les noms d'hôtes comme `127.0.0.1` et `localhost`, ou des URL sensibles comme `/admin`.
- Le filtre peut être contourné avec les techniques suivantes:
	- Utiliser une autre représentation IP de `127.0.0.1` : `2130706433`, `017700000001` ou `127.1`.
	- Enregistrer son propre nom de domaine qui pointe vers `127.0.0.1` (par exemple `spoofed.burpcollaborator.net`).
	- Masquer les chaînes bloquées par encodage d'URL ou en variant la casse.
	- Fournir une URL contrôlée par l'attaquant qui redirige vers l'URL cible, en essayant différents codes de redirection et différents protocoles. Passer d'une URL `http:` à `https:` lors de la redirection a permis de contourner certains filtres anti-SSRF.
### SSRF avec filtres d'entrée basés sur une liste blanche
- Certaines applications n'autorisent que les entrées correspondant à une liste blanche de valeurs.
- Le filtre peut chercher une correspondance au début de l'entrée ou à l'intérieur.
- Il peut être contourné en exploitant des incohérences dans l'analyse des URL, car les fonctionnalités de la spécification sont souvent négligées quand l'analyse et la validation sont faites de façon ad hoc.
- Ce filtre peut être contourner de la manière suivante:
	- Identifiants avant le nom d'hôte, avec le caractère `@` : `https://expected-host:fakepassword@evil-host`
	- Fragment d'URL, avec le caractère `#` : `https://evil-host#expected-host`
	- Hiérarchie DNS : placer la valeur attendue dans un nom DNS complet que l'on contrôle : `https://expected-host.evil-host`
	- Encodage d'URL pour semer la confusion dans l'analyse. Utile surtout si le code du filtre traîne les caractères encodés différemment du code qui effectue la requête HTTP.
	- Double encodage : certains serveurs décodent de manière récursive, ce qui peut créer d'autres divergences.
### Contournement des filtres SSRF via une vulnérabilités de redirection ouverte
- Condition: l'application dont les URL sont autorisées contient une redirection ouverte et l'API qui effectue la requête HTTP côté serveur prend en charge les redirections.
- L'attaquant construit alors une URL qui satisfait le filtre mais aboutit à une requête redirigée vers la cible interne voulue.
- Exemple:
	- L'URL `/product/nextProduct?currentProductId=6&path=http://evil-user.net` renvoie une redirection vers `http://evil-user.net`.
- Exploitation:
	- L'attaquant envoie `stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin`.
- Pourquoi ça marche?
	- L'application vérifie d'abord que l'URL `stockApi` est sur un domaine autorisé, ce qui est le cas.
	- Elle interroge ensuite cette URL, ce qui déclenche la redirection ouverte.
	- Elle suit la redirection et envoie une requête vers l'URL interne choisie par l'attaquant.

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
### Lab: SSRF with blacklist-based input filter
This lab has a stock check feature which fetches data from an internal system.
To solve the lab, change the stock check URL to access the admin interface at `http://localhost/admin` and delete the user `carlos`.
The developer has deployed two weak anti-SSRF defenses that you will need to bypass.

1. Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
2. Change the URL in the `stockApi` parameter to `http://127.0.0.1/` and observe that the request is blocked.
3. Bypass the block by changing the URL to: `http://127.1/`
4. Change the URL to `http://127.1/admin` and observe that the URL is blocked again.
5. Obfuscate the "a" by double-URL encoding it to %2561 to access the admin interface and delete the target user.
### Lab: SSRF with filter bypass via open redirection vulnerability
This lab has a stock check feature which fetches data from an internal system.
To solve the lab, change the stock check URL to access the admin interface at `http://192.168.0.12:8080/admin` and delete the user `carlos`.
The stock checker has been restricted to only access the local application, so you will need to find an open redirect affecting the application first.

1. Visit a product, click "Check stock", intercept the request in Burp Suite, and send it to Burp Repeater.
2. Try tampering with the `stockApi` parameter and observe that it isn't possible to make the server issue the request directly to a different host.
3. Click "next product" and observe that the `path` parameter is placed into the Location header of a redirection response, resulting in an open redirection.
4. Create a URL that exploits the open redirection vulnerability, and redirects to the admin interface, and feed this into the `stockApi` parameter on the stock checker:
    `/product/nextProduct?path=http://192.168.0.12:8080/admin`
5. Observe that the stock checker follows the redirection and shows you the admin page.
6. Amend the path to delete the target user:
    `/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos`
## Liens
- [[Command-injection]]
- [[XXE-injection]]
- [[Web-cache-poisoning]]
