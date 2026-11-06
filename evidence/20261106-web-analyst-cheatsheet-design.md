# Conception de la refonte de Payloads-cheatsheet.md

Date : 2026-11-06. Auteur : web-analyst. Statut : conception, la cheat sheet n'a pas été modifiée.
Cible : /home/capss/web-bscp/BSCP-Vault/04-Essential-skills/Payloads-cheatsheet.md (163 lignes, 11 thèmes).
Règle de contenu : uniquement des payloads et techniques génériques, jamais de solution ni de déroulé d'un lab précis.

## 1. Diagnostic de l'existant

Constats vérifiés sur le fichier lu en entier.

1. Listes plates à puces : chaque thème est une suite de puces de format `payload - explication`, sans sous-groupe. La section SQL injection fait 47 puces (lignes 117 à 163) sans sous-titre : détection, colonnes, extraction, aveugle, erreur, délai et hors bande sont mélangés.
2. Payloads noyés dans des phrases : lignes 126, 129, 130, 138 à 141, 147, 152, 154, 158 à 160 contiennent plusieurs payloads dans une même puce, séparés par des virgules ou des points-virgules. Impossible de copier un payload sans sélectionner à la main.
3. Marqueurs de contexte mélangés : le moteur ("Oracle :", "Microsoft", "hors Oracle") est tantôt en début de puce, tantôt au milieu, tantôt entre parenthèses ; l'état ("à vérifier", "non rejoué") est dans des phrases (lignes 157, 162).
4. Pas de séparation par moteur ni par système : SQL injection mélange Oracle, Microsoft, PostgreSQL, MySQL, SQLite ; Command injection mélange Unix et Windows (ligne 114) ; Path traversal mélange Unix et Windows.
5. Placeholders non homogènes : `cond` (lignes 142 à 159), `COLLAB` (160), `BURP-COLLABORATOR-SUBDOMAIN` (94, 95, 113), `EXPLOIT-SERVER` (43, 48), `<suffixe>`, `<ip aléatoire>`, `<fichier>`, `<motclé><année>`, `§x§`, `§a§`, `X`. Aucune légende.
6. Tirets cadratins : présents sur les lignes 8 à 105 (sections Path traversal à File upload), les sections Command injection et SQL injection utilisent déjà le tiret normal. Le style est donc incohérent.
7. Chaînes de flèches et flèches : lignes 26 (`?id=123 → ?id=124`), 39 (substitutions `o`→`0`), 75 et 77 (`→`). Contraire à la règle de style.
8. Doublons : `X-Forwarded-For` apparaît dans Authentication (40) et sera dans Access control ; `COLLAB` / Collaborator apparaît dans SSRF, Command injection, SQL injection et Authentication ; `Referer` dans Access control et SSRF ; Path traversal (`..%252f`) recoupe File upload (`..%2f`) ; les fichiers cibles de path traversal et le chemin `/etc/passwd` reviennent ailleurs.
9. Absence d'indication d'état : seule la section SQL injection dit que `TO_CHAR(1/0)` est le seul payload rejoué (ligne 157). Ailleurs, aucune ligne ne dit si elle a été éprouvée. Les revues SSRF et Command injection relèvent des lignes reconstituées de mémoire (SSRF lignes 92 et 95).
10. Contenu qui n'est pas un payload : méthodologie Burp (Intruder, Grep - Match, Resource pool, macro, Proxy history), script Turbo Intruder de 19 lignes dans Authentication, remarques de procédure ("tester sous un compte différent"). Cela noie les payloads.
11. Éléments liés à un lab précis (à retirer ou généraliser lors de la réécriture) : `wiener:peter`, `carlos:`, `stockApi=...`, `/administrator-panel-<suffixe>`, `Update email`, `weliketoshop`, `/home/carlos/secret`, `storeID`, `Gifts'--`, `username=carlos&current-password=...`, `stay-logged-in` avec détail Intruder. La règle est : technique générique uniquement, noms de paramètres remplacés par `PARAM`.
12. Accès difficile : pas de table des matières, pas d'ancres, titres de niveau 3 sans titre de niveau 1 ou 2, front matter minimal. Le lien vers la note détaillée est noyé en dernière puce.
13. Erreurs de contenu à corriger à l'occasion : ligne 75 `Cookie: account=` contre `verify=carlos` dans le journal (question à poser à l'utilisateur, voir revue Authentication partie 5) ; ligne 114 "Unix seulement" pour `;` et saut de ligne, inexact pour PowerShell (revue Command injection).
14. Un seul fichier de 163 lignes avec des puces très longues (jusqu'à 700 caractères) : le rendu Obsidian force un défilement horizontal ou un retour à la ligne qui masque les espaces significatifs.

## 2. Structure cible

### 2.1 Ordre des thèmes

Ordre par fréquence d'usage en examen puis par proximité technique. Chaque thème garde le nom de sa note parente.

1. SQL injection (la plus volumineuse, mise en premier pour avoir ses sous-sections moteur)
2. Command injection
3. Path traversal
4. File upload
5. SSRF
6. Access control
7. Authentication
8. Multi-factor authentication
9. Outils Burp transverses (annexe : Intruder, Turbo Intruder, Collaborator, Resource pool). C'est le seul endroit pour la méthodologie d'outil.

Les thèmes futurs (XSS, XXE, désérialisation, etc.) suivent le même gabarit et s'ajoutent avant l'annexe.

### 2.2 Squelette du fichier

```
---
tags: [bscp, essentiel, payloads]
statut: en cours
mis-a-jour: 2026-11-06
---
# Fiche de payloads

(1) Légende (placeholders, états, encodage)       <- une seule fois, en haut
(2) Table des matières (liens internes vers les ancres)
(3) Un titre de niveau 2 par thème, gabarit identique
(4) Annexe : outils Burp
```

La légende vient avant la table des matières car la table est courte et la légende s'applique à tout. Les deux tiennent sur un écran.

### 2.3 Table des matières

Liens internes Obsidian vers les titres, un par thème, sur une ligne chacun, avec les sous-sections moteur en retrait.

```
- [[#SQL injection]]
  - [[#SQL injection - Détection]]
  - [[#SQL injection - Union]]
  - [[#SQL injection - Aveugle booléen]]
  - [[#SQL injection - Error-based]]
  - [[#SQL injection - Délai]]
  - [[#SQL injection - Hors bande]]
- [[#Command injection]]
...
```

Règle d'ancre : les titres de niveau 3 sont préfixés par le nom du thème (`SQL injection - Error-based`) pour que les ancres `[[#...]]` soient uniques dans le fichier. Sans cela, deux sections "Détection" entrent en collision.

Pour le repérage rapide pendant un lab : Obsidian affiche aussi le plan (panneau Outline) à partir des titres ; la hiérarchie 2 puis 3 uniquement (pas de niveau 4) garde ce plan lisible. Donc : niveau 2 = thème, niveau 3 = technique ou famille, jamais plus profond. Les moteurs et systèmes ne sont PAS des titres mais des lignes de tableau (voir 2.5).

### 2.4 Gabarit identique par thème

Chaque thème (titre de niveau 2) suit exactement cet ordre. Une section vide est conservée avec "Aucun payload consigné" pour que le gabarit reste identique.

```
## NOM DU THÈME

Note détaillée : [[Note-parente]] ; fiches : [[Fiche-1]], [[Fiche-2]]

**Quand l'utiliser** : une phrase (condition d'application).
**Détection** : une à trois lignes (quoi envoyer, quoi observer).

### NOM DU THÈME - Famille ou technique A

(tableau Situation / Payload / Signal attendu / État)  puis blocs de code
...
```

Contraintes :
- "Quand l'utiliser" et "Détection" : texte court, jamais de payload dans ces phrases (les payloads de détection vont dans la première sous-section "Détection" sous forme de tableau et de bloc de code).
- Le renvoi vers la note détaillée est la deuxième ligne de chaque thème, sous le titre, pour être visible sans défiler.
- Pas de liste de liens à la fin.

### 2.5 Mise en forme d'une sous-section : tableau d'index puis bloc de code

Problème à résoudre : un tableau Markdown est bon pour retrouver (Situation, Signal), mais une cellule de tableau altère le payload (le tube `|` casse les colonnes, les backticks dans un payload cassent le code en ligne, Obsidian peut interpréter `[[`, `#`, `%`, `*`, `_`). Un bloc de code est bon pour copier mais ne se range pas en colonnes.

Solution retenue : séparer l'index et la charge utile.

1. Le tableau contient Situation, Signal attendu, Remarque et État, plus une colonne Réf. avec un identifiant court (par exemple `SQL-E1`).
2. Juste en dessous, UN bloc de code par moteur ou système, une ligne par payload, chaque ligne précédée d'un commentaire court `-- SQL-E1` (ou `# CMD-3`) qui renvoie au tableau. Le payload est donc copiable sans rien d'autre que la ligne.

Alternative écartée : mettre un payload dans chaque cellule avec échappement du tube. Dans les tableaux Obsidian, le tube s'échappe en `\|` y compris dans un code en ligne ; mais le rendu affiche parfois la barre oblique inverse en mode lecture et le copier-coller la récupère. Trop risqué pour des payloads où chaque caractère compte. Cette variante reste acceptable uniquement pour un petit tableau (moins de trois payloads) sans tube ; sinon utiliser la solution retenue.

Règles de blocs de code :
- Une ligne = un payload prêt à coller après remplacement des placeholders. Jamais deux payloads sur une ligne.
- Pas de ligne coupée (pas de retour à la ligne manuel dans un payload). Si un payload est long, il reste sur une ligne ; Obsidian propose le défilement horizontal du bloc, ce qui ne modifie pas le contenu.
- Langage du bloc : `sql` pour SQL (coloration utile, ne modifie pas le texte), `bash` pour Unix, `text` pour tout le reste (URL, en-têtes, JSON). Pas de `html` pour les charges XSS futures (Obsidian pourrait rendre certains aperçus) : utiliser `text`.
- Un bloc par moteur ou système, précédé d'une ligne en gras qui nomme le moteur : `**Oracle**`, `**Microsoft SQL Server**`, `**PostgreSQL**`, `**MySQL et MariaDB**`, `**SQLite**`, `**Unix**`, `**Windows (cmd)**`, `**Windows (PowerShell)**`. Ce nom en gras est le "sous-titre" de moteur ; il n'est pas un titre Markdown, donc pas dans le plan, mais il est visible au défilement.
- Le commentaire de référence suit la syntaxe du langage du bloc : `--` suivi d'un espace en SQL, `#` en bash, `#` en `text` (le caractère `#` y est juste du texte, jamais copié par erreur car il est placé en fin de ligne précédente ; voir ci-dessous).
- Pour éviter qu'une référence soit copiée avec le payload : la ligne de référence est une ligne SÉPARÉE au-dessus du payload, pas en fin de ligne. Un double-clic de sélection de ligne prend uniquement la ligne du payload. Exemple dans le modèle (partie 3).

Caractères à surveiller dans les blocs :
- Espace final significatif (par exemple `-- ` suivi d'un espace en MySQL) : Obsidian et les éditeurs suppriment parfois les espaces de fin de ligne. Solution : écrire le payload avec `-- -` (tiret final qui protège l'espace) ou `--+` en URL, et le signaler dans la colonne Remarque. Règle générale : tout payload SQL MySQL se termine par `-- -` et non par `-- `.
- Guillemets typographiques : interdire dans les blocs (Obsidian ne les crée pas, mais un copier-coller depuis un navigateur ou un traitement de texte le peut). L'écrivain vérifie par recherche des caractères `‘ ’ “ ”` dans le fichier fini.
- Tiret cadratin et flèches : interdits partout, y compris dans les blocs (un payload n'en contient jamais ; si c'est le cas, c'est une erreur de recopie).

### 2.6 Solution pour le tube et les autres caractères qui cassent les tableaux

Règle : aucun payload n'apparaît dans une cellule de tableau. La colonne "Payload" demandée dans la structure initiale est remplacée par la colonne "Réf." qui pointe vers la ligne du bloc de code. Les cellules du tableau n'ont que du texte descriptif sans caractère spécial.

Si un symbole doit être cité dans une cellule (par exemple "séparateur tube"), l'écrire en toutes lettres ("tube", "esperluette", "point-virgule") ou utiliser l'entité HTML `&#124;` pour le tube. L'entité est reconnue par Obsidian dans les tableaux en lecture et en aperçu en direct. Ne pas utiliser `\|` dans les cellules : le rendu et la copie divergent.

Autres caractères à traiter :
- `[[` dans un payload : dans un bloc de code, aucune interprétation (sûr).
- `#` : dans un bloc de code, sûr. Dans une cellule, écrire "dièse".
- `$`, `{{ }}` : sûrs en bloc de code. Désactiver les extensions de maths si le rendu pose problème (`$` dans un texte courant). Dans les cellules, éviter les `$` ; utiliser des mots.
- `<` `>` en texte hors bloc : peuvent être pris pour du HTML. Dans les cellules, écrire en toutes lettres ("chevron ouvrant").
- `%` : sûr dans tous les cas.

### 2.7 Légende unique en haut

Un seul tableau, placé avant la table des matières. Format de placeholder unique : MAJUSCULES entre chevrons, sans trait d'union dans le nom, par exemple `<COLLAB>`.

| Placeholder | Signification | Remplacer par |
|---|---|---|
| `<COLLAB>` | Sous-domaine d'interaction hors bande | Sous-domaine Burp Collaborator ou domaine de l'exploit server |
| `<COND>` | Condition booléenne testée | Expression vraie ou fausse selon le test, par exemple `1=1` |
| `<TABLE>` | Nom de table | Nom découvert par énumération |
| `<COLONNE>` | Nom de colonne | Nom découvert par énumération |
| `<UTILISATEUR>` | Nom de compte ciblé | Valeur du test |
| `<N>` | Nombre ou position | Entier à faire varier |
| `<HOTE>` | Nom d'hôte ou adresse | Cible interne ou externe selon le test |
| `<PARAM>` | Nom du paramètre vulnérable | Nom réel dans la requête |
| `<CHEMIN>` | Chemin de fichier ou d'URL | Valeur du test |
| `<VALEUR>` | Donnée libre | Valeur du test |
| `<DELAI>` | Durée du délai en secondes | 10 par défaut |

Autres éléments de légende dans le même bloc :
- États : voir 2.8.
- Position Intruder : `§X§` conserve ce format Burp exact, jamais renommé.
- Encodage : une ligne qui rappelle que dans une URL, `+` = espace, `%2b` = plus, `%23` = dièse, `%3b` = point-virgule, `%26` = esperluette, `%7c` = tube, `%0a` = saut de ligne, `%00` = octet nul.
- Règle : tout nouveau placeholder est d'abord ajouté à la légende, jamais défini en ligne. `COLLAB`, `BURP-COLLABORATOR-SUBDOMAIN`, `EXPLOIT-SERVER` et `COLLABORATOR` deviennent tous `<COLLAB>`.

### 2.8 Convention d'état

Colonne "État" dans chaque tableau, trois valeurs seulement, avec symbole texte (pas d'émoji) :

| Valeur | Sens |
|---|---|
| `REJOUE` | Éprouvé par l'utilisateur dans un lab, avec la date au format AAAA-MM (par exemple `REJOUE 2026-09`) |
| `A VERIFIER` | Source fiable mais jamais rejoué ; ne pas s'y fier sans test de contrôle |
| `ANCIEN` | Dépend de la version du moteur ; vérifier d'abord la version |

Règles :
- Défaut à l'écriture : tout est `A VERIFIER`, sauf ce que le journal des labs confirme. L'écrivain ne passe une ligne à `REJOUE` que si une note du coffre (journal de lab) ou l'utilisateur le dit. D'après la cheat sheet actuelle, seul `TO_CHAR(1/0)` sur Oracle est explicitement rejoué dans la partie erreur.
- L'état est aussi rappelé dans le commentaire du bloc de code de façon discrète : la ligne de référence porte un suffixe `?` quand l'état est `A VERIFIER`, par exemple `-- SQL-E3 ?`. Ainsi l'état se voit dans le bloc sans ouvrir le tableau.
- Les lignes dont la revue dit "lab non réalisé, reconstitué de mémoire" (SSRF `localhost:80%2523@...`, Shellshock) sont `A VERIFIER` ; elles ne sont pas retirées mais marquées.
- Après chaque lab réalisé, l'utilisateur (pas l'agent) bascule la ligne.

### 2.9 Gestion de l'affichage dans Obsidian

- Mode de lecture et aperçu en direct : les blocs de code ont un bouton copier au survol, qui copie le bloc entier. Conséquence : pour que "copier" donne un seul payload, il faudrait un bloc par payload. Compromis retenu : un bloc par moteur avec 3 à 8 lignes ; l'utilisateur sélectionne la ligne (triple clic). Si l'utilisateur préfère le bouton copier, passer en un bloc par payload pour les thèmes les plus utilisés (décision à lui poser, voir partie 6).
- Ne pas utiliser de callouts ni de blocs repliables pour les payloads (le contenu replié ne ressort pas dans la recherche rapide en aperçu).
- Largeur de tableau : au maximum 4 colonnes (Réf., Situation, Signal attendu, État) plus Remarque en dernière position si nécessaire (5 colonnes max). Des cellules courtes (moins de 80 caractères) évitent le défilement.
- Recherche : la recherche Obsidian (Ctrl+Maj+F) trouve le texte des blocs. Convention de mots-clés en tête de ligne de référence, par exemple `-- SQL-E1 error-based oracle`, pour retrouver en tapant "error oracle". Les mots-clés sont en minuscules, sans accents.
- Ligne de référence en commentaire du langage : dans un bloc `text`, utiliser `#` ; il n'y a pas d'interprétation.
- Pas de caractère Markdown dans les noms de titres (`#`, `|`, crochets) pour garder des ancres sûres.
- Aucun tiret cadratin, aucune flèche, aucune abréviation maison : vérification finale par recherche (voir partie 5, contrôles).
- Menu Outline à droite ou commande "Aller au titre" : titres niveau 2 et 3 seulement.

## 3. Modèle concret : thème SQL injection, sous-section Error-based

Le modèle montre l'ouverture du thème (gabarit) puis la sous-section Error-based seule. L'écrivain applique ce patron à l'identique à toutes les sous-sections et à tous les thèmes. Contenu issu de la cheat sheet actuelle (lignes 142 à 157) ; la revue Error-based SQLi (fichier 20261106-101558...) est manquante au moment de cette rédaction, l'écrivain doit la relire pour compléter.

Début du modèle (tout ce qui suit la ligne de séparation est à écrire dans la cheat sheet).

---

## SQL injection

Note détaillée : [[SQL-injection]] ; fiches : [[SQL-injection-Union-Attacks]], [[Blind-SQL-injection]], [[Error-based SQL Injection]]

**Quand l'utiliser** : un paramètre, un cookie ou un en-tête est inséré dans une requête SQL.
**Détection** : envoyer une apostrophe seule, puis deux apostrophes ; une erreur puis un retour à la normale confirme l'injection.

### SQL injection - Error-based

**Quand l'utiliser** : la page n'affiche pas le résultat de la requête, mais une erreur de base de données est visible (verbeuse) ou son absence se remarque (conditionnelle).

| Réf. | Situation | Signal attendu | Remarque | État |
|---|---|---|---|---|
| SQL-E1 | Oracle, erreur conditionnelle | Erreur de division par zéro si la condition est vraie, page normale sinon | Choisir `<COND>` avec soin ; ne jamais atteindre un UPDATE ou un DELETE | REJOUE 2026-09 |
| SQL-E2 | Microsoft, erreur conditionnelle | Erreur de division par zéro si la condition est vraie | | A VERIFIER |
| SQL-E3 | PostgreSQL, erreur conditionnelle | Erreur de division par zéro si la condition est vraie | | A VERIFIER |
| SQL-E4 | PostgreSQL, erreur verbeuse | Message `invalid input syntax for type integer` suivi de la valeur | Si le message est tronqué, vider la valeur d'origine du cookie | A VERIFIER |
| SQL-E5 | MySQL et MariaDB, erreur verbeuse | Message `XPATH syntax error` suivi de la valeur | Environ 32 caractères affichés ; extraire par tranches avec `SUBSTRING` | A VERIFIER |
| SQL-E6 | Microsoft, erreur verbeuse | Message `Conversion failed when converting the nvarchar value` suivi de la valeur | | A VERIFIER |
| SQL-E7 | Oracle, erreur verbeuse | Message `ORA-31011: XML parsing failed` avec la valeur dans le détail | | A VERIFIER |

Reconnaissance du moteur par le message d'erreur :

| Fragment du message | Moteur probable |
|---|---|
| `XPATH syntax error` ou `Subquery returns more than 1 row` | MySQL |
| `invalid input syntax for type integer` | PostgreSQL |
| `ORA-` | Oracle |
| `Conversion failed` ou `Divide by zero error` | Microsoft SQL Server |

**Oracle**

```sql
-- SQL-E1
' || (SELECT CASE WHEN (<COND>) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '
-- SQL-E7 ?
' AND 1=(SELECT UPPER(XMLTYPE(CHR(60)||CHR(58)||(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' AND ROWNUM=1)||CHR(62))) FROM dual)-- -
```

**Microsoft SQL Server**

```sql
-- SQL-E2 ?
' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/0 ELSE 1 END)-- -
-- SQL-E6 ?
' AND 1=CONVERT(int,(SELECT TOP 1 <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>'))-- -
```

**PostgreSQL**

```sql
-- SQL-E3 ?
' AND 1=(SELECT CASE WHEN (<COND>) THEN 1/(SELECT 0) ELSE 1 END)-- -
-- SQL-E4 ?
' AND 1=CAST((SELECT <COLONNE> FROM <TABLE> LIMIT 1) AS int)-- -
```

**MySQL et MariaDB**

```sql
-- SQL-E5 ?
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT <COLONNE> FROM <TABLE> WHERE username='<UTILISATEUR>' LIMIT 1)))-- -
```

Fin du modèle. Notes d'application pour l'écrivain, hors cheat sheet :
- Dans le modèle, la colonne État du tableau et le suffixe ` ?` du bloc doivent toujours concorder.
- Le modèle ci-dessus ne reprend pas toutes les variantes de la cheat sheet actuelle (UPDATEXML, double requête MySQL, IF MySQL, UTL_INADDR, CTXSYS, SQLite, STRING_AGG, FOR XML PATH, `::int`) : l'écrivain les ajoute en lignes SQL-E8 et suivantes, avec l'état `ANCIEN` pour la double requête MySQL et `A VERIFIER` pour les autres. SQLite a son propre bloc `**SQLite**`.
- Les noms `users`, `username`, `password`, `administrator` qui figurent dans la cheat sheet actuelle sont remplacés par `<TABLE>`, `<COLONNE>`, `<UTILISATEUR>`, sauf quand le nom fait partie de l'explication (le nom `users` est le plus courant et peut rester dans une remarque).
- Le choix `-- -` en fin de payload est uniforme pour MySQL, PostgreSQL, Microsoft ; Oracle accepte `--` seul. L'uniformité évite le piège de l'espace final supprimé.
- `' AND 1=` : les payloads de la cheat sheet actuelle sont écrits avec des espaces ; la version `+` pour URL n'est pas dupliquée. La légende d'encodage explique la conversion. Une seule forme par payload.

## 4. Plan de migration par thème (consignes à l'écrivain)

Pour chacun des 8 thèmes : reprendre les lignes actuelles, les classer dans des sous-sections, retirer la méthodologie d'outil vers l'annexe, remplacer les noms propres de lab par des placeholders, appliquer la convention d'état.

| Thème | Sous-sections proposées (niveau 3) | Systèmes ou moteurs à séparer | Éléments à sortir ou corriger |
|---|---|---|---|
| SQL injection | Détection ; Commentaires et conditions ; Union (colonnes, types, extraction, concaténation, version, schéma) ; Aveugle booléen ; Error-based ; Délai ; Hors bande | Oracle, Microsoft SQL Server, PostgreSQL, MySQL et MariaDB, SQLite | Retirer `Gifts'--` et `administrator'--` en tant que cas de lab (garder la technique générique du commentaire et du contournement de connexion avec `<UTILISATEUR>`) ; déplacer Intruder vers l'annexe ; ajouter un tableau de concaténation par moteur |
| Command injection | Détection ; Séparateurs ; Délai ; Redirection de sortie ; Hors bande ; Contournement de filtre | Unix, Windows (cmd), Windows (PowerShell) | Corriger la ligne 114 (`;` valable en PowerShell) ; remplacer `storeID` par `<PARAM>` ; ajouter la table des encodages |
| Path traversal | Séquences simples ; Séquences imbriquées ; Encodages ; Chemin absolu ; Préfixe ; Extension et octet nul ; Fichiers cibles | Unix, Windows | Mettre la table des fichiers cibles en deux blocs ; la liste Intruder dédiée va à l'annexe ; le rappel curl `--path-as-is` est une remarque d'outil |
| File upload | Web shells ; Extension ; Type MIME ; Contenu et signature ; Fichier de configuration ; Traversée dans le nom ; Concurrence | PHP (seul moteur consigné) | Les techniques des labs non réalisés (revue File upload partie 4) ne sont ajoutées qu'après que l'utilisateur les a faites, une par une : voir partie 6 |
| SSRF | Cibles locales ; Balayage interne ; Représentations d'adresse ; Contournement liste noire ; Contournement liste blanche ; Redirection ouverte ; Aveugle ; Métadonnées cloud | Aucun (séparer par famille) | Remplacer `stockApi` par `<PARAM>` ; marquer `A VERIFIER` les lignes reconstituées de mémoire |
| Access control | Accès direct ; Paramètres et cookies ; Identifiants (IDOR) ; Méthode HTTP ; En-têtes de réécriture ; Variantes de chemin ; Referer | Aucun | Remplacer les flèches ; retirer `/administrator-panel-<suffixe>` en tant que cas de lab, garder `/admin` |
| Authentication | Énumération de comptes ; Mots de passe ; Contournement de blocage ; Réinitialisation ; Cookie persistant ; HTTP Basic | Aucun | Sortir le script Turbo Intruder vers l'annexe ; retirer `wiener:peter`, `carlos:` ; remplacer les flèches `o`→`0` par des mots |
| Multi-factor authentication | Contournement de l'étape ; Brute force du code ; Réponse manipulable | Aucun | Résoudre la question `account=` contre `verify=` avec l'utilisateur avant d'écrire |

Annexe Burp : Intruder (positions, Sniper, Pitchfork, Cluster bomb, payload processing, Grep - Match, Grep - Extract), Resource pool à une requête simultanée, Turbo Intruder (script), Collaborator (Poll now), macros et règles de session. Gabarit identique mais sans colonne Moteur.

## 5. Contrôles de qualité avant livraison par l'écrivain

Commandes à lancer sur le fichier réécrit (recherches simples, aucune modification) :

```bash
cd /home/capss/web-bscp/BSCP-Vault/04-Essential-skills
grep -n "—" Payloads-cheatsheet.md          # tirets cadratins : doit être vide
grep -n "–" Payloads-cheatsheet.md          # tirets demi-cadratins : doit être vide
grep -n "→\|->\|=>" Payloads-cheatsheet.md  # flèches hors blocs de code : doit être vide
grep -n "COLLAB\b\|BURP-COLLABORATOR\|EXPLOIT-SERVER" Payloads-cheatsheet.md   # doit montrer seulement <COLLAB>
grep -n "wiener\|carlos\|weliketoshop\|stockApi\|storeID\|Gifts" Payloads-cheatsheet.md   # doit être vide
grep -nP "[‘’“”]" Payloads-cheatsheet.md   # guillemets typographiques : doit être vide
```

Contrôle de l'état : chaque ligne de tableau a une valeur d'état ; le nombre de suffixes ` ?` dans les blocs égale le nombre de lignes `A VERIFIER`.
Contrôle de couverture : chaque payload de la version actuelle se retrouve dans la version neuve (comparer avec `git diff` ou la copie de sauvegarde du dépôt) ; aucune perte.
Contrôle d'affichage : ouvrir dans Obsidian, vérifier qu'aucun tableau n'est cassé et que le plan (Outline) montre les niveaux 2 et 3.

## 6. Questions pour l'utilisateur (avant ou pendant la réécriture)

1. Préfère-t-il le bouton copier d'Obsidian (un bloc de code par payload, plus long) ou la sélection d'une ligne dans un bloc par moteur (retenu par défaut) ?
2. `Cookie: account=carlos` ou `verify=carlos` pour le thème MFA (divergence entre cheat sheet et journal) ?
3. Veut-il que les techniques des labs non réalisés (File upload : extensions, `.htaccess`, polyglotte, concurrence ; SSRF labs 6 et 7 ; Command injection Practitioner) soient dans la cheat sheet dès maintenant, ou ajoutées après chaque lab ? La revue File upload recommande l'ajout après chaque lab pour ne pas donner d'indice. Recommandation : ne pas les ajouter maintenant ; garder dans chaque thème une section "À ajouter après lab" vide.
4. Quels payloads peut-il confirmer comme rejoués ? Sans cette information, tout est `A VERIFIER` sauf `TO_CHAR(1/0)`.

## 7. Sources de contenu à intégrer

Sections "à reporter dans la cheat sheet" des revues du dossier /home/capss/web-bscp/evidence/.

| Revue | Section à lire | Thème visé | Disponible |
|---|---|---|---|
| 20261106-web-analyst-access-control-review.md | Partie 4 (ligne 151) | Access control | Oui |
| 20261106-web-analyst-command-injection-review.md | Partie 4 (ligne 138), plus la correction sur `;` et PowerShell | Command injection | Oui |
| 20261106-web-analyst-file-upload-review.md | Partie 4 (ligne 159), à ne verser qu'après labs | File upload | Oui |
| 20261106-web-analyst-path-traversal-review.md | Partie 4 (ligne 219) | Path traversal | Oui |
| 20261106-web-analyst-ssrf-review.md | Partie 4 (ligne 188) | SSRF | Oui |
| 20261106-web-analyst-authentication-review.md | Partie 5 (ligne 268), section nommée "À reporter dans la cheat sheet" ; la partie 4 (ligne 190) est un catalogue de payloads | Authentication et Multi-factor authentication | Oui (présent lors de la vérification finale) |
| 20261106-web-analyst-sql-injection-review.md | Section à reporter dans la cheat sheet (voir grep ci-dessous) | SQL injection | Oui (présent lors de la vérification finale) |
| 20261106-101558-web-analyst-error-based-sqli-review.md | Partie à reporter | SQL injection - Error-based | Absent du dossier de travail, récupérable dans git |

Note : le fichier d'évidence `20261106-101558-web-analyst-error-based-sqli-review.md` apparaît comme supprimé dans l'état git du dépôt (statut D) ; il se récupère par `git show HEAD:evidence/20261106-101558-web-analyst-error-based-sqli-review.md` s'il n'est pas régénéré.

Points d'attention récurrents dans les revues : tout ce qui est reporté est `A VERIFIER` ; ne pas dupliquer ce que la cheat sheet contient déjà ; remplacer tirets cadratins et flèches ; respecter l'accord sur les labs non réalisés.

## 8. Vérification finale de la présence des revues

Résultat de `ls /home/capss/web-bscp/evidence` au moment de la rédaction : 
- Présents : access-control, authentication, command-injection, file-upload, path-traversal, sql-injection, ssrf.
- Absent du dossier de travail : 20261106-101558-web-analyst-error-based-sqli-review.md (supprimé, statut D dans git). Il existe dans le dernier commit : `git -C /home/capss/web-bscp show HEAD:evidence/20261106-101558-web-analyst-error-based-sqli-review.md`.
- Je n'ai pas relu le contenu des revues Authentication et SQL injection en détail ; seul l'intitulé de leur section à reporter a été repéré.
