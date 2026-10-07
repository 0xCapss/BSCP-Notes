---
tags:
  - bscp
  - server-side
  - path-traversal
niveau: apprentice
statut: en cours
---
# Path traversal

## En bref
- L'attaque path traversal (ou directory traversal) est une vulnérabilité web qui permet à un attaquant de lire des fichiers arbitraires sur le serveur qui héberge l'application, voire d'en écrire dans certains cas.
- Ces fichiers peuvent contenir du code applicatif, des identifiants de back-end ou des fichiers système sensibles.

## Types et variantes
- Lecture de fichier arbitraire (le cas le plus courant) contre écriture de fichier arbitraire (via upload, export ou sauvegarde de configuration), cette dernière ayant un impact potentiellement plus critique (exécution de code).
- Convention de chemin Unix (`../`) contre Windows (`../` et `..\` valides toutes les deux).
- Traversal non filtré (séquence simple suffisante) contre traversal filtré, nécessitant une technique de contournement.

| Type | Idée clé | Note | Niveau |
| --- | --- | --- | --- |
| Cas simple | `../../../etc/passwd` | [[Path-traversal-cas-simple]] | Apprentice |
| Chemin absolu | `filename=/etc/passwd` | [[Path-traversal-chemin-absolu]] | Practitioner |
| Séquences imbriquées et encodage | `....//`, `%252e%252e%252f` | [[Path-traversal-sequences-et-encodage]] | Practitioner |
| Préfixe et extension | préfixe imposé, `%00.png` | [[Path-traversal-prefixe-et-extension]] | Practitioner |

## Comment détecter
- Repérer les paramètres ou fonctionnalités qui manipulent un nom de fichier ou un chemin (paramètres nommés filename, file, path, doc, template, page, image, download, include, ou valeurs contenant une extension de fichier).
- Injecter une séquence de traversal simple visant un fichier connu et observable (`../../../etc/passwd` sur Linux, `..\..\..\windows\win.ini` sur Windows) et vérifier si le contenu du fichier apparaît dans la réponse.
- Si le retour n'est pas directement visible, chercher des signaux indirects : différence de code de statut, de message d'erreur, de taille de réponse ou de temps de réponse entre un chemin existant et un chemin inexistant.
- Si la séquence simple est filtrée, tester méthodiquement chaque technique de contournement (encodage simple, double encodage, doubled characters, chemin absolu, null byte) pour identifier précisément le filtre en place.
- Automatiser les tests avec Burp Intruder et une liste de payloads de traversal sur les paramètres suspects.
- Ne pas se limiter aux fonctionnalités de lecture : vérifier aussi les fonctionnalités d'upload, d'export ou de sauvegarde de configuration qui pourraient permettre une écriture de fichier arbitraire.


## Comment exploiter (principe)
1. Identifier le paramètre qui désigne un fichier.
2. Tester `../../../etc/passwd` (ou `..\..\..\windows\win.ini`) et lire le corps de la réponse.
3. Si la séquence est bloquée, essayer dans l'ordre : chemin absolu, séquences imbriquées, encodage simple puis double, préfixe imposé, octet nul avant l'extension.
4. Confirmer la lecture avec un fichier connu, puis cibler des fichiers utiles (configuration, code source, clés).

Le détail de chaque technique est dans les notes de la table ci-dessus.


## Pièges et points d'attention BSCP
- Bien distinguer un simple filtrage de la sous-chaîne `../` (contournable par doubled characters) d'une validation par canonicalisation robuste (résolution du chemin absolu puis vérification qu'il reste dans le répertoire autorisé), beaucoup plus difficile à contourner.
- Ne pas s'arrêter au premier payload qui échoue : les labs combinent souvent plusieurs filtres empilés (décodage, puis suppression de séquences, puis vérification d'extension), chacun nécessitant sa propre technique de contournement.
- Vérifier si le serveur applique un décodage simple ou double avant de choisir entre `%2e%2e%2f` et `%252e%252e%252f`.
- Sur les labs de type "validation que le chemin commence par le dossier attendu", tester le remplacement complet du paramètre par un chemin absolu.
- Sur les labs de type "validation de l'extension de fichier", tester la technique du null byte avant l'extension attendue.
- Tester systématiquement la lecture et l'écriture/upload, cette dernière ayant un impact potentiellement plus critique (exécution de code).
- Si l'environnement cible n'est pas connu, tester les deux conventions de chemin (Unix et Windows).
- Attention aux faux positifs : un message d'erreur générique ne prouve pas la lecture effective du fichier ; exiger un signal explicite (contenu du fichier retourné, ou différence de comportement mesurable et reproductible).

## Prévention
- Éviter de transmettre des entrées utilisateur directement à une API de système de fichiers ; privilégier un mapping indirect (identifiant ou clé associé côté serveur à un chemin fixe, sans jamais exposer le chemin réel à l'utilisateur).
- Si un nom de fichier doit néanmoins être accepté depuis l'utilisateur, le valider strictement par liste blanche (extension autorisée, absence de séparateurs de chemin, correspondance à un pattern ancré) ; à défaut d'une liste blanche stricte, vérifier au minimum que les données ne contiennent que des caractères autorisés.
- Après validation, concaténer les données au répertoire de base puis canoniser le chemin obtenu avec l'API du système de fichiers de la plateforme, et vérifier que le résultat commence bien par le répertoire de base attendu avant tout accès au fichier (exemple de code ci-dessous).
- Ne jamais se fier à un simple retrait de la sous-chaîne `../` (vulnérable aux doubled characters) ni implémenter sa propre logique de canonicalisation ; utiliser les fonctions fournies par le langage ou le framework.
- Appliquer le principe de moindre privilège sur le compte du processus serveur (accès en lecture/écriture restreint aux seuls répertoires nécessaires) en défense en profondeur.
- Journaliser et alerter sur les tentatives contenant des séquences de traversal ou des caractères suspects dans les paramètres de fichier.
- Exemple de code Java simple pour valider le chemin final :
  ```java
  File file = new File(BASE_DIRECTORY, userInput);
  if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
      // process file
  }
  ```


## Labs PortSwigger
- [x] Apprentice ✅ 2026-09-14
- [x] Practitioner ✅ 2026-09-16

## Journal des labs

### Lab 1 - Traversée de chemin, cas simple
*File path traversal, simple case* - Apprentice - note : [[Path-traversal-cas-simple]]

Ce lab contient une vulnérabilité de traversée de chemin dans l'affichage des images de produits. Pour le résoudre, récupérez le contenu du fichier `/etc/passwd`.

1. Interceptez dans Burp la requête qui charge une image de produit.
2. Remplacez le paramètre `filename` par `../../../etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier `/etc/passwd`.

**Mes notes de résolution**
- Cible de l'époque : `0a7400fb036c87db800efd7300b4005f.web-security-academy.net`.
- Point d'entrée vulnérable : `GET /image?filename=`, utilisé normalement pour servir les images produit (`filename=53.jpg`).
- Le payload `../../../../etc/passwd` a fonctionné directement, sans contournement : aucun filtre sur `../`, aucune canonicalisation, aucune restriction d'extension. La réponse a un `Content-Type: image/jpeg` alors que le corps est le texte de `/etc/passwd`.
- Oracle de comportement relevé avant l'exploitation : fichier existant, HTTP 200 ; fichier inexistant (`filename=doesnotexist.jpg`), HTTP 400 avec un corps JSON `"No such file"`. Utile pour confirmer une lecture réussie même quand le contenu n'est pas directement lisible.

### Lab 2 - Séquences de traversée bloquées, contournement par chemin absolu
*File path traversal, traversal sequences blocked with absolute path bypass* - Practitioner - note : [[Path-traversal-chemin-absolu]]

L'application bloque les séquences de traversée mais traite le nom de fichier fourni comme relatif au répertoire de travail par défaut. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par le chemin absolu `/etc/passwd`, sans aucune séquence de traversée.
3. Constatez que la réponse contient le contenu du fichier.

### Lab 3 - Séquences de traversée retirées de façon non récursive
*File path traversal, traversal sequences stripped non-recursively* - Practitioner - note : [[Path-traversal-sequences-et-encodage]]

L'application retire les séquences de traversée de l'entrée avant de l'utiliser. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `....//....//....//etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier : après le retrait d'une occurrence de `../`, il reste une séquence valide.

### Lab 4 - Séquences de traversée retirées avec un décodage d'URL superflu
*File path traversal, traversal sequences stripped with superfluous URL-decode* - Practitioner - note : [[Path-traversal-sequences-et-encodage]]

L'application bloque les entrées qui contiennent des séquences de traversée, puis décode l'entrée une seconde fois. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `..%252f..%252f..%252fetc/passwd` (le `/` est double-encodé).
3. Constatez que la réponse contient le contenu du fichier.

### Lab 5 - Validation du début du chemin
*File path traversal, validation of start of path* - Practitioner - note : [[Path-traversal-prefixe-et-extension]]

L'application exige que le paramètre `filename` commence par le répertoire de base attendu. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit et notez la valeur de `filename` (par exemple `/var/www/images/12.jpg`).
2. Remplacez-la par `/var/www/images/../../../etc/passwd`.
3. Constatez que la réponse contient le contenu du fichier.

### Lab 6 - Validation de l'extension avec contournement par octet nul
*File path traversal, validation of file extension with null byte bypass* - Practitioner - note : [[Path-traversal-prefixe-et-extension]]

L'application exige que le paramètre `filename` se termine par une extension de fichier attendue. Récupérez `/etc/passwd`.

1. Interceptez la requête qui charge une image de produit.
2. Remplacez `filename` par `../../../etc/passwd%00.png`.
3. Constatez que la réponse contient le contenu du fichier.

**Mes notes de résolution**
- Labs Practitioner résolus le 2026-09-16. Il a fallu tester plusieurs techniques de « Comment exploiter » (traversée classique, chemin absolu, préfixe, octet nul, encodage) selon le filtre rencontré. Détails d'endpoint et de requête à compléter si besoin de les rejouer.

## Mes notes
- 

## Liens
- [[Path-traversal-cas-simple]]
- [[Path-traversal-chemin-absolu]]
- [[Path-traversal-sequences-et-encodage]]
- [[Path-traversal-prefixe-et-extension]]
- [[File-upload]]
- [[Information-disclosure]]
- [[Command-injection]]
- [[Payloads-cheatsheet]]
