# Revue documentaire - notes Authentification (2026-11-06)

Périmètre : trois notes de `/home/capss/web-bscp/BSCP-Vault/01-Server-side/Authentification/` :
- `Authentication.md` (221 lignes)
- `Autres-mecanismes-authentification.md` (121 lignes)
- `Multi-factor-authentication.md` (77 lignes)

Méthode : lecture seule des trois notes, lecture rapide des en-têtes des dix fiches de `Types/` (toutes au statut "à faire") et de la section Authentication de `/home/capss/web-bscp/BSCP-Vault/04-Essential-skills/Payloads-cheatsheet.md` pour éviter les doublons. Aucune note modifiée, aucune requête envoyée, rien rejoué. Le catalogue de la section 3 est donc entièrement documentaire : tout ce qui n'a pas été rejoué par l'utilisateur est marqué "à vérifier".

## 0. État des labs d'après le contenu des notes

Les 14 labs PortSwigger d'authentification sont tous présents sous forme rédigée dans les journaux.

Journal de `Authentication.md` (12 labs) :

| Lab | Titre | Niveau | Contenu dans la note | Case cochée |
| --- | --- | --- | --- | --- |
| 1 | Énumération via réponses différentes | Apprentice | déroulé rédigé | oui (Apprentice) |
| 2 | Énumération via réponses subtilement différentes | Practitioner | déroulé rédigé | oui (Practitioner) |
| 3 | Énumération via temps de réponse | Practitioner | déroulé rédigé | oui |
| 4 | Blocage par IP défaillant | Practitioner | déroulé rédigé | oui |
| 5 | Énumération via verrouillage de compte | Practitioner | déroulé rédigé, étape 4 floue | oui |
| 6 | Plusieurs identifiants par requête | Expert | déroulé rédigé | non (Expert décoché) |
| 7 | Force brute du cookie persistant | Practitioner | déroulé rédigé | oui |
| 8 | Craquage hors ligne | Practitioner | déroulé rédigé | oui |
| 9 | Logique de réinitialisation défaillante | Apprentice | déroulé rédigé | oui |
| 10 | Contournement simple de la 2FA | Apprentice | déroulé rédigé | oui |
| 11 | Logique défaillante de la 2FA | Practitioner | déroulé rédigé | oui |
| 12 | Contournement de la 2FA par force brute | Expert | déroulé rédigé | non (Expert décoché) |

Journal de `Autres-mecanismes-authentification.md` : lab 1 (empoisonnement via middleware, Practitioner) et lab 2 (force brute via modification du mot de passe, Practitioner), tous deux rédigés, cases Apprentice et Practitioner cochées.

`Multi-factor-authentication.md` : aucun journal propre, renvoi vers les labs 10 à 12 de `Authentication.md`. Expert décoché.

Constats sur l'état des labs :
- Réalisés (déroulé rédigé et case cochée) : 12 labs sur 14, soit les labs 1 à 5 et 7 à 11 de `Authentication.md` (donc hors labs 6 et 12) plus les deux labs de `Autres-mecanismes-authentification.md`. Je ne peux pas distinguer une solution rédigée par l'utilisateur d'une paraphrase de la solution officielle : plusieurs énoncés (labs 4, 5, 9) reprennent presque mot pour mot la formulation PortSwigger.
- Ambigus : les labs 6 et 12 (Expert) ont un déroulé rédigé mais la case Expert est décochée dans `Authentication.md` et dans `Multi-factor-authentication.md`. Deux lectures possibles : labs non faits et déroulé copié depuis la solution, ou labs faits mais case oubliée. Question à poser à l'utilisateur, pas à trancher ici. Recommandation : ne rien ajouter sur ces deux labs, laisser les notes en l'état tant que l'utilisateur n'a pas confirmé. Ne pas cocher à sa place.
- Non réalisés sans contenu rédigé : aucun lab. Il n'y a donc aucune zone vide à remplir ; rien à proposer de ce côté. La fiche `MFA-reponse-et-secours.md` n'a pas de lab associé (aucun lab PortSwigger dédié) : rien à faire.
- Les dix fiches `Types/` sont au statut "à faire" : leur remplissage est un choix de l'utilisateur, pas un manque des trois notes.

## 1. Note `Authentication.md`

### 1.1 Ce qui manque ou est flou

Concepts :
- Pas de définition de la session après authentification (cookie de session, identifiant prévisible, fixation de session). Le titre de la fiche parle de "mécanisme" mais seule l'étape de connexion est traitée.
- Pas de distinction explicite entre brute force classique, credential stuffing (réutilisation de fuites) et password spraying (un mot de passe, plusieurs comptes). Cette dernière est pourtant la contre-mesure naturelle au verrouillage par compte.
- Pas de mention des identifiants par défaut ni des politiques de mot de passe faibles comme point de départ, alors que le cheatsheet les cite.
- La phrase "Deux origines de vulnérabilité" (ligne 22) est bonne mais sans exemple rattaché à chaque origine.

Méthodologie de détection :
- Ligne 36 : "comparer les réponses" sans dire quoi mesurer précisément (longueur, statut, texte, temps, en-têtes comme `Set-Cookie`, redirection). Pas de mention de la comparaison via Burp Comparer ni du tri par colonne.
- Aucune mention du bruit de mesure pour le temps de réponse : répéter, prendre la médiane, éviter d'enchaîner sur la même connexion.
- Ne dit pas comment lire le lab 5 : à quel moment un mot de passe est jugé valide.
- Pas de mention des autres sources d'énumération : formulaire d'inscription (nom déjà pris), formulaire de mot de passe oublié (message différent), comptes en-têtes de réponse.

Cas par variante :
- Le tableau des types renvoie aux fiches mais ne donne pas, par variante, le signal attendu ni la limite.
- La variante HTTP Basic n'a aucun lab dans le journal : c'est normal (pas de lab dédié), mais la note ne le dit pas.

Limites et contre-mesures :
- Ligne 60 : "manipulation d'en-têtes" sans nommer les en-têtes (`X-Forwarded-For`, `X-Real-IP`, `X-Client-IP`, `True-Client-IP`), alors que le cheatsheet le fait.
- Pas de mention du CAPTCHA, des délais progressifs ni des limites de débit par compte (contre-mesure citée seulement dans l'autre note).
- Pas de limite côté attaquant : rate limiting par session, comptes réels verrouillés en production, risque de déni de service sur un compte victime (verrouillage volontaire).

Pièges BSCP :
- Le tableau et les pièges ne rappellent pas que BSCP impose un temps limité : privilégier d'abord les signaux peu coûteux (différence de réponse) avant le temps de réponse.
- Les labs 3 et 4 reposent sur `X-Forwarded-For` ; la note ne dit pas que ce n'est pas universel (voir limites ci-dessus).
- Le mode Intruder à choisir par variante (Sniper, Pitchfork, Cluster bomb) n'est expliqué que dans les étapes de labs, jamais en synthèse.

### 1.2 Superflu, redondant ou hors sujet

- Remarques de rédaction laissées dans le texte : ligne 47 "Détails dans les notes de la table ci-dessus." (renvoi de navigation, à retirer ou à fondre dans le tableau). Lignes 48-49 : deux lignes vides consécutives.
- Redondances internes : la ligne 38 (détecter), la ligne 54 (pièges) et la note MFA répètent la même vérification de l'état de session avant la deuxième étape. La ligne 37 (détecter) et la ligne 52 (pièges) répètent la question du périmètre du seuil de blocage.
- Redondance avec `Autres-mecanismes-authentification.md` : la section "Prévention" de cette dernière (lignes 54-62 : énumération, force brute) traite le périmètre de `Authentication.md` ; la prévention de `Authentication.md` couvre les mêmes points plus brièvement. À fusionner dans une seule.
- Le journal contient 12 labs dont 6 appartiennent à d'autres notes (cookie persistant, craquage hors ligne, réinitialisation, 2FA x3). C'est un choix d'organisation, mais l'autre note renvoie vers celle-ci (ligne 113) et la note MFA aussi (ligne 65), ce qui rend la navigation circulaire.
- Section "Mes notes" (lignes 203-206) : la distinction authentification et autorisation est une définition générale, sa place est dans "En bref" avec le lien [[Access-control]].
- Liens : [[JWT-attacks]], [[OAuth]], [[Business-logic]] sont plausibles (autres mécanismes d'authentification et logique), mais non justifiés dans le texte. Ils sont à garder avec une phrase d'explication ou à retirer. Il manque [[HTTP-Host-header]] (existe dans `03-Advanced/`), pertinent pour l'empoisonnement de la réinitialisation. [[Payloads-cheatsheet]] : légitime.
- Image `![](Authentication.png)` : le fichier existe dans `01-Server-side/assets/`, pas dans le dossier de la note. Si le rendu échoue, utiliser le chemin complet. Aucune légende ni description de l'image.
- Frontmatter : `niveau: apprentice` alors que le contenu va jusqu'au niveau Expert.
- Fautes et style : "brute force" et "brute-force" alternent sans règle. Les symboles de coche avec émoji (lignes 64-65) sont une convention de l'utilisateur, à ne pas modifier.

### 1.3 Plan de réécriture

1. Frontmatter : niveau couvrant Apprentice à Expert, statut inchangé tant que les Expert ne sont pas confirmés.
2. "En bref" : ajouter la distinction authentification et autorisation (déplacer depuis "Mes notes").
3. "Types et variantes" : ajouter une colonne "signal attendu" au tableau ; ajouter une courte entrée sur brute force, credential stuffing et password spraying.
4. "Comment détecter" : transformer en liste de mesures (statut, longueur, texte, temps, `Set-Cookie`), ajouter les autres sources d'énumération (inscription, mot de passe oublié).
5. "Pièges" : dédoublonner avec "Comment détecter" ; nommer les en-têtes de substitution d'adresse IP ; ajouter une ligne sur le choix du mode Intruder.
6. "Prévention" : fusionner avec celle d'`Autres-mecanismes-authentification.md` et ne garder ici que celle de la page de connexion (énumération, force brute, Basic).
7. Journal : ne pas toucher aux déroulés des labs (rédigés et cochés). Pour les labs 6 et 12, rien à ajouter tant que l'utilisateur n'a pas dit s'ils sont faits. Option de rangement : laisser le journal ici (aucun risque pour les déroulés) et se contenter de corriger la navigation circulaire par un tableau récapitulatif en tête de journal.
8. Liens : ajouter une phrase d'explication par lien, ajouter [[HTTP-Host-header]], supprimer les lignes vides doubles.

## 2. Note `Autres-mecanismes-authentification.md`

### 2.1 Ce qui manque ou est flou

Concepts :
- La note ne définit pas ce que sont les "fonctionnalités annexes" au-delà de la liste (maintien de connexion, mot de passe oublié, changement de mot de passe).
- Pas de mention de l'inscription ni de la vérification d'e-mail, ni de la fonctionnalité "questions de sécurité".

Méthodologie de détection :
- Réinitialisation : pas de test sur la prévisibilité du token (longueur, alphabet, horodatage, séquentialité), l'expiration, l'usage unique, ni la liaison du token à un compte précis.
- Pas de test des paramètres dupliqués (`email` en double) ni du paramètre `email` modifiable dans le corps.
- Maintien de connexion : "chercher un motif prévisible" reste vague ; il faut dire comment comparer deux comptes (même longueur ? même préfixe ? même suffixe ?), tester l'encodage (base64, hexadécimal), le haché (MD5, SHA1, SHA256) et la présence d'un sel.
- Modification du mot de passe : pas de test de CSRF ni du changement sans mot de passe actuel.

Cas par variante :
- La table des types est correcte mais sans signal ni limite.
- Le poisoning de réinitialisation est présenté seulement via `X-Forwarded-Host`. Les variantes `Host` modifié, double `Host`, ligne de requête absolue, `X-Forwarded-Server`, `X-Host`, `Forwarded`, ne sont qu'en piège (ligne 36).

Limites et contre-mesures :
- Aucune limite pour le poisoning : il faut qu'un clic de la victime ou un client e-mail qui précharge le lien, ou un lien non validé par le serveur ; sur une cible réelle, le domaine d'envoi est souvent fixé par configuration (le lab utilise un middleware qui fait confiance à l'en-tête).
- Les contre-mesures spécifiques (token à haute entropie, lié au compte, usage unique, expiration courte, URL absolue construite depuis la configuration et non depuis l'en-tête) sont absentes de la prévention, qui est générique.

Pièges BSCP :
- Ligne 37 : explication confuse du token "volé côté attaquant" et du token "légitime". Reformuler : le journal d'accès de l'exploit server capte le token généré pour la victime ; ce token est ensuite utilisé sur le formulaire légitime reçu par e-mail pour son propre compte. L'idée est bonne, la phrase brouille les rôles.
- Pas de rappel que le compte de la victime est verrouillé si les deux nouveaux mots de passe coïncident (information utile présente seulement dans le journal du lab 2, ligne 102).

### 2.2 Superflu, redondant ou hors sujet

- Ligne 32 : phrase incorrecte. Elle dit de soumettre un mot de passe actuel correct puis de forcer le paramètre `current-password` : en réalité le mot de passe actuel est la valeur à deviner. Texte à corriger : "soumettre un mot de passe actuel candidat avec deux nouveaux mots de passe différents ; le message `New passwords do not match` apparaît seulement si le candidat est correct". Faute de genre dans la même ligne : "une oracle" devient "un oracle".
- Ligne 33 : "Vol de token via XSS" est mal nommé : c'est le vol du cookie persistant par XSS, qui relève de [[Auth-maintien-connexion]] (lab 8 de `Authentication.md`). Il n'est pas un mécanisme d'attaque de cette note en tant que tel.
- Ligne 14 : redondante avec la ligne 9.
- Lignes 13 et 14 d'"Types et variantes" ne décrivent pas de variante.
- Section "Prévention" (lignes 42-70) : contient des sujets sans lien avec les fonctionnalités annexes (politique de mot de passe, 2FA par SMS, énumération). Les blocs MFA appartiennent à `Multi-factor-authentication.md`, les blocs énumération et force brute à `Authentication.md`. La liste imbriquée à quatre niveaux est difficile à lire.
- Ligne 61 : "Idéalement, exiger un CAPTCHA" sans nuance (les CAPTCHA sont contournables).
- Section "Mes notes" (ligne 113) : renvoi circulaire vers `Authentication.md`. À remplacer par un tableau d'aiguillage des labs.
- Liens : [[Access-control]] est peu justifié ; il manque [[HTTP-Host-header]].
- Journal : le lab 1 contient un "Compte perso : wiener:peter" ; le lab 2 contient un lien externe vers la liste de mots de passe PortSwigger, c'est acceptable et utile. Ne rien changer aux déroulés.

### 2.3 Plan de réécriture

1. "En bref" : conserver, enrichir avec la liste des fonctionnalités concernées (inscription, vérification d'e-mail).
2. "Types et variantes" : supprimer les deux phrases génériques, garder le tableau et ajouter une colonne "signal attendu".
3. "Comment détecter" : ajouter les tests de token (entropie, expiration, usage unique, liaison au compte), de paramètres dupliqués, de CSRF sur le changement de mot de passe.
4. "Comment exploiter" : corriger la ligne 32, déplacer la ligne 33 vers `Auth-maintien-connexion`, ajouter les variantes d'en-têtes de Host comme sous-liste.
5. "Pièges" : reformuler la ligne 37, ajouter le verrouillage lié aux nouveaux mots de passe identiques.
6. "Prévention" : ne garder que les points propres aux fonctionnalités annexes (token, URL absolue côté serveur, mot de passe actuel obligatoire, CSRF) ; déplacer le reste.
7. Journal : inchangé.
8. Liens : ajouter [[HTTP-Host-header]], justifier ou retirer [[Access-control]].

## 3. Note `Multi-factor-authentication.md`

### 3.1 Ce qui manque ou est flou

Concepts :
- Pas d'explication du fonctionnement des codes TOTP (durée de 30 secondes, 6 chiffres, tolérance d'une ou deux fenêtres), ni de la différence entre OTP envoyé par SMS ou e-mail et code généré localement. Ces détails déterminent l'espace de recherche et la fenêtre de brute force.
- Les mécanismes de secours et l'inscription du second facteur (qui peut l'activer, le désactiver ou le réinitialiser) ne sont décrits qu'en une ligne chacun.

Méthodologie de détection :
- Ligne 25 : "tester s'il existe une limitation" sans dire comment (compter les essais avant coupure, vérifier si la limitation est par session, par compte ou par IP, vérifier si une nouvelle connexion remet le compteur à zéro).
- Pas de test d'expiration chiffré (attendre N minutes puis rejouer).
- Pas de test de réutilisation d'un code d'une session à l'autre ni de l'invalidation du code quand un nouveau code est demandé.
- La section "Comment exploiter" est une liste de quatre étapes sans signal attendu.

Cas par variante :
- Les quatre variantes du tableau sont décrites, mais les fiches associées sont vides (statut "à faire"). La note elle-même n'indique ni signal ni limite par variante.
- Absence de mention des flux parallèles : réinitialisation de mot de passe qui connecte directement, connexion via OAuth ou API mobile qui sautent le second facteur. Ils sont cités en détection (ligne 29) mais pas dans le tableau.

Limites et contre-mesures :
- Pas de limite côté attaquant : le brute force sur le code se heurte à la limitation par session (cas du lab Expert) et à l'expiration.
- Prévention complète, mais sans lien explicite entre chaque contre-mesure et la variante qu'elle neutralise.

Pièges BSCP :
- Le rappel que les macros et la gestion de session de Burp sont indispensables pour la variante Expert n'est que dans le journal de `Authentication.md`.
- Le temps nécessaire (10 000 requêtes en une seule connexion simultanée) n'est pas évalué : à noter pour la gestion du temps à l'examen.

### 3.2 Superflu, redondant ou hors sujet

- Section "Mes notes" : une ligne vide avec un tiret seul (ligne 68). À renseigner ou supprimer.
- Section "Journal des labs" : un simple renvoi vers `Authentication.md` ; aucun contenu propre. Un tableau d'aiguillage (lab, niveau, fiche, case) serait plus utile.
- Ligne 48 (piège sur la "two-step verification") répète le paragraphe d'ouverture (ligne 10).
- Les lignes 45 et 46 (deux pièges sur l'absence de limitation et l'absence d'expiration) sont utiles mais la ligne 45 mélange deux idées ; la distinction entre les deux causes (liaison d'identité, limitation) est celle du lab 11, qui n'est pas cité.
- Le cheatsheet cite `Cookie: account=carlos` pour la liaison d'identité, alors que le journal parle de `verify=carlos` : incohérence à lever à l'occasion de la mise à jour du cheatsheet (je ne le modifie pas).
- Liens : [[Access-control]] est plausible (accès direct à une page protégée) mais non justifié.
- Frontmatter : `niveau: apprentice` alors que la note va jusqu'à Expert.

### 3.3 Plan de réécriture

1. Frontmatter : niveau couvrant Apprentice à Expert.
2. "En bref" : ajouter un paragraphe sur les formes de code (TOTP, SMS, e-mail, push) et leur espace de recherche.
3. "Comment détecter" : ajouter la mesure de la limitation (nombre d'essais, portée, remise à zéro), de l'expiration et de l'invalidation.
4. "Comment exploiter" : ajouter une colonne "signal attendu" au tableau (par exemple page protégée accessible avec le statut 200, `302` vers la page de compte) et une colonne "limite".
5. "Pièges" : ajouter le coût en requêtes de la variante Expert, la nécessité de la macro, le rapport entre les labs 11 et 12 (deux causes qui se combinent).
6. "Prévention" : rattacher chaque contre-mesure à la variante qu'elle neutralise.
7. "Journal des labs" : remplacer le renvoi par un tableau d'aiguillage ; ne rien ajouter sur le lab 12 tant que l'utilisateur n'a pas confirmé son état.
8. "Mes notes" : supprimer la ligne vide ou y mettre une remarque personnelle.

## 4. Catalogue de payloads et techniques

Légende de la colonne Statut :
- "Journal" : la technique figure dans un déroulé de lab rédigé par l'utilisateur (labs cochés). Pas rejouée par moi.
- "À vérifier" : technique générale ou non présente dans un déroulé ; à valider sur lab ou environnement autorisé avant de la considérer acquise. Les techniques issues des labs 6 et 12 (cases Expert décochées) sont aussi marquées "à vérifier" tant que l'utilisateur n'a pas confirmé.

### 4.1 Énumération de noms d'utilisateur

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Tri par longueur de réponse dans Intruder | Message différent entre nom invalide et mot de passe invalide | Une longueur isolée | Inefficace si la longueur est identique | Journal (lab 1) |
| Grep - Extract sur le message d'erreur | Réponses presque identiques | Une réponse avec un point final manquant | Sensible à la casse et à la ponctuation | Journal (lab 2) |
| Mot de passe très long (environ 100 caractères) + temps de réponse | Hachage exécuté seulement si le nom existe | Temps nettement plus long | Bruit réseau, blocage par IP à contourner | Journal (lab 3) |
| Verrouillage comme oracle (Cluster bomb avec Null payloads x5) | Verrouillage après N essais | Message de verrouillage sur un seul nom | Temps d'attente du déverrouillage ; sur cible réelle, verrouille de vrais comptes | Journal (lab 5) |
| Nom déjà pris à l'inscription | Formulaire d'inscription | Message "déjà utilisé" | Dépend de l'existence de l'inscription | À vérifier |
| Message différent sur "mot de passe oublié" | Formulaire de réinitialisation | "Si ce compte existe" absent ou différent | Souvent uniformisé | À vérifier |
| Différence de statut ou en-tête (`Set-Cookie`, `Location`) | Réponses identiques en texte | Écart sur un en-tête | Dépend de l'implémentation | À vérifier |

### 4.2 Brute force de mots de passe

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Sniper sur `password` avec nom fixé | Nom connu | `302` | Bloqué si limitation | Journal (lab 1) |
| Pitchfork pour associer en-tête et mot de passe | Blocage par IP contournable | `302` | Dépend de la confiance accordée à l'en-tête | Journal (lab 3) |
| Password spraying (un mot de passe, plusieurs noms) | Verrouillage par compte | `302` sur un nom | Verrouillage par IP ou global | À vérifier |
| Credential stuffing (couples issus de fuites) | Réutilisation de mots de passe | `302` | Légalité et périmètre : seulement avec autorisation | À vérifier |
| Mutations prévisibles (`Password1!`, `<motclé><année>!`, leet) | Politique de complexité | `302` | Dépend de la population | À vérifier |

### 4.3 Contournement de la limitation de débit et du verrouillage

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| `X-Forwarded-For` variable | L'IP est lue dans l'en-tête | Plus de blocage en changeant la valeur | N'existe pas partout (proxy qui réécrit) | Journal (lab 3) |
| Alterner un login valide et un essai (Pitchfork, une requête simultanée) | Compteur remis à zéro après succès | Aucun blocage sur les essais | Nécessite un compte valide | Journal (lab 4) |
| Tableau JSON de mots de passe dans une seule requête | Corps JSON, limitation par requête | Connexion réussie en une requête | Lié à l'implémentation du serveur | Journal (lab 6), à vérifier : lab Expert décoché |
| Variantes d'en-têtes (`X-Real-IP`, `X-Client-IP`, `True-Client-IP`, `Forwarded`) | `X-Forwarded-For` ignoré | Idem | Idem | À vérifier (déjà citées dans le cheatsheet) |
| Variations de casse ou d'espace du nom (`Carlos`, `carlos `) | Compteur indexé sur la chaîne brute | Le compteur ne s'incrémente pas | Rare | À vérifier |

### 4.4 Cookies persistants

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Décoder le cookie (base64, hexadécimal) | Case "Stay logged in" | Structure lisible `utilisateur:haché` | Cookie signé ou aléatoire | Journal (lab 7) |
| Payload processing : Hash MD5, Add prefix `carlos:`, Encode Base64 | Cookie `base64(nom:md5(mot de passe))` | Texte de page connectée (`Update email`) | Dépend de l'algorithme et de l'absence de sel | Journal (lab 7) |
| XSS stocké pour récupérer le cookie, puis craquage hors ligne | Faille XSS dans les commentaires | Requête de la victime dans le journal d'accès de l'exploit server | Cookie `HttpOnly` bloque l'accès JavaScript ; haché salé | Journal (lab 8) |
| Comparer des cookies de comptes différents | Pas de modèle connu | Motif commun | Nécessite plusieurs comptes | À vérifier |

### 4.5 Réinitialisation et modification du mot de passe

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Supprimer le token de l'URL et du corps | Token non revalidé | La requête réussit encore | Dépend de l'implémentation | Journal (lab 9 de `Authentication.md`) |
| Changer `username` dans le corps de la réinitialisation | Compte pris dans le corps | Mot de passe de la victime changé | Idem | Journal (lab 9) |
| `X-Forwarded-Host` vers l'exploit server | Lien construit depuis l'en-tête | Requête de la victime dans le journal d'accès, avec token | Victime qui clique ; domaine fixé en configuration | Journal (lab 1 d'`Autres-mecanismes`) |
| `X-Forwarded-Server`, `X-Host`, `Forwarded`, double `Host` | `X-Forwarded-Host` filtré | Même signal | Peu fiable | À vérifier |
| Paramètre `email` dupliqué ou liste séparée par une virgule | Envoi du lien à une adresse de l'attaquant | Lien reçu pour un autre compte | Rare, dépend du serveur | À vérifier |
| Token prévisible (horodatage, séquence, MD5 de l'identifiant) | Génération faible | Tokens successifs proches | Demande d'obtenir plusieurs tokens | À vérifier |
| Oracle `New passwords do not match` sur `current-password` | Champ masqué `username` modifiable | Un seul message différent | Verrouillage si les deux nouveaux mots de passe sont identiques | Journal (lab 2 d'`Autres-mecanismes`) |

### 4.6 HTTP Basic

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Décoder `Authorization: Basic` | Réponse `401` avec `WWW-Authenticate: Basic` | `nom:mot de passe` en clair | Sans HTTPS ni HSTS, interception possible | À vérifier (aucun lab) |
| Brute force de l'en-tête (Intruder avec Encode Base64) | Pas de limitation | `200` au lieu de `401` | Limitation, blocage par IP | À vérifier |

### 4.7 Contournement de la deuxième étape (MFA)

| Technique | Contexte | Signal attendu | Limites | Statut |
| --- | --- | --- | --- | --- |
| Aller directement à l'URL post-connexion (`/my-account`) | Session déjà "connectée" après l'étape 1 | Page de compte affichée | Corrigé si le serveur revérifie l'étape 2 | Journal (lab 10) |
| Modifier le cookie `verify=carlos` puis forcer `mfa-code` | Identité portée par un cookie | `302` après le bon code | Code limité en tentatives, durée de vie | Journal (lab 11) |
| Brute force 0000-9999 avec macro et gestion de session | Déconnexion après deux échecs | `302` | Coût, hasard (plusieurs passes) | Journal (lab 12), à vérifier : lab Expert décoché |
| Changer `"verified":false` en `"verified":true` | Décision côté client | Accès accordé | Rare | À vérifier |
| Codes de secours, cookie de confiance de l'appareil | Mécanismes secondaires | Accès sans code | Dépend de l'implémentation | À vérifier |
| Flux parallèles (réinitialisation, OAuth, API mobile) | Second facteur non exigé partout | Session ouverte sans code | Dépend de l'application | À vérifier |
| Réutilisation d'un code après usage ou après expiration | Code non invalidé | Connexion avec un ancien code | Dépend de l'implémentation | À vérifier |

## 5. À reporter dans la cheat sheet (`04-Essential-skills/Payloads-cheatsheet.md`)

État actuel : la section Authentication du cheatsheet (lignes 37-52) couvre déjà la majorité des techniques consignées dans le journal. Les ajouts ci-dessous sont des éléments qui n'y figurent pas encore. Tout ce qui est marqué "à vérifier" l'est aussi dans le catalogue ; à reporter seulement après vérification par l'utilisateur, sinon l'indiquer dans la ligne.

Ajouts courts, prêts à copier :
- `X-Real-IP`, `X-Client-IP`, `True-Client-IP`, `Forwarded: for=<ip>` - variantes d'en-têtes pour contourner le blocage par IP (à vérifier, sauf `X-Forwarded-For`)
- Un mot de passe, plusieurs noms (password spraying) - contourne le verrouillage par compte (à vérifier)
- Comparer longueur, statut, message, temps, `Set-Cookie` entre nom invalide et mot de passe invalide - énumération d'utilisateurs (rappel de méthode)
- Inscription ("nom déjà utilisé") et mot de passe oublié (message différent) - autres sources d'énumération (à vérifier)
- `echo -n 'user:pass' | base64` et `echo '<valeur>' | base64 -d` - encoder et décoder `Authorization: Basic` et les cookies persistants (à vérifier)
- `echo -n 'motdepasse' | md5sum` - calculer le haché d'un candidat pour le cookie persistant
- Paramètre dupliqué `email=victime@x.tld&email=attaquant@x.tld` - envoi du lien de réinitialisation à l'attaquant (à vérifier)
- Réinitialisation : demander plusieurs tokens successifs et comparer - tokens prévisibles (à vérifier)
- Mode Intruder par cas : Sniper pour un paramètre, Pitchfork pour deux listes alignées (en-tête de substitution d'IP + candidat), Cluster bomb pour toutes les combinaisons (verrouillage), une requête simultanée pour l'alternance de compte valide et d'essai
- MFA : tester `/my-account` directement, puis une valeur d'identité modifiable (le journal utilise `Cookie: verify=carlos`), puis le brute force du code avec macro
- MFA : codes de secours, cookie de confiance de l'appareil, flux de réinitialisation et API mobile - chemins parallèles à tester (à vérifier)

Corrections à signaler dans le cheatsheet existant (non modifié) :
- Ligne 75 du cheatsheet : `Cookie: account=carlos` ne correspond pas au journal qui utilise `verify=carlos`. Vérifier avec l'utilisateur laquelle des deux formes reporter.
- Plusieurs lignes du cheatsheet utilisent le symbole de flèche (par exemple ligne 75 : `account=carlos` flèche `account=victim-user`) ; la règle de style du projet l'interdit. À corriger par l'utilisateur ou par `report-writer`.

## 6. Questions pour l'utilisateur

1. Les labs 6 (plusieurs identifiants par requête) et 12 (2FA par force brute) sont-ils réalisés ? Leur déroulé est rédigé, mais les cases Expert sont décochées dans `Authentication.md` et `Multi-factor-authentication.md`. Tant que la réponse n'est pas donnée, je recommande de ne rien ajouter sur ces deux labs et de laisser ces notes dans leur état actuel.
2. Les déroulés des labs sont-ils de ta propre rédaction ? Plusieurs énoncés suivent de très près la solution officielle de PortSwigger. Si ce sont des paraphrases, c'est sans conséquence pour la note, mais à connaître pour le statut "Journal" de la section 4.
