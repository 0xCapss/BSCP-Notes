---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# Error-based SQL Injection

## En bref
- Cette technique d'injection désigne les cas où les messages d'erreurs permettent d'extraire ou de déduire des données sensibles de la base, même dans un contexte blind.
- Pour cela 2 approches:
	- **Erreurs conditionnelles:** On peut parfois induire en erreur une réponse d'erreur spécifique selon le résultat d'une exporession booléenne. L’exploitation est la même que pour les réponses conditionnelles vues précédemment.
	- **Message d'erreurs verbeux**: On peut parfois déclencher des messages d'erreurs qui affichent les données renvoyées par la requête. Cela transforme une blind SQLi en injection visible.


## Types et variantes
.

## Comment détecter


## Comment exploiter (principe)
- Il est souvent possible d’induire une réponse différente selon qu’une erreur SQL se produit ou non.
- On modifie la requête pour qu’elle provoque une erreur de base de données uniquement si la condition injectée est vraie.
- Une erreur non gérée provoque très souvent une différence dans la réponse de l’application, comme un message d’erreur.
- Ainsi cette différence permet de déduire si la condition injectée est vraie ou fausse.

### Erreurs conditionnelles avec `CASE`

- Deux requêtes sont envoyées tour à tour, avec ces valeurs pour le cookie `TrackingId` :
    - `xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a`
    - `xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a`
- Le mot-clé `CASE` teste une condition et renvoie une expression différente selon qu’elle est vraie ou fausse.

**Résultat de chaque requête**

- Première valeur : la condition `1=2` est fausse, `CASE` renvoie `'a'`, donc aucune erreur.
- Seconde valeur : la condition `1=1` est vraie, `CASE` renvoie `1/0`, ce qui provoque une erreur de division par zéro.
- Si l’erreur change la réponse HTTP de l’application, on peut en déduire si la condition injectée est vraie.

**Extraction de données**

- On récupère les données en testant un caractère à la fois : `xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`
- Si l’erreur se produit, le premier caractère du mot de passe de `Administrator` est supérieur à `m`. Sinon, il ne l’est pas.
- On répète ensuite avec d’autres comparaisons et d’autres positions, comme pour les réponses conditionnelles.

**Point pratique** (connaissance générale, non issue de votre texte)

- Sur Oracle, `1/0` doit souvent être remplacé par `TO_CHAR(1/0)` et la requête nécessite `FROM DUAL`


## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [x] Practitioner ✅ 2026-11-02
- [ ] Expert

## Journal des labs
### Blind SQL injection avec erreurs conditionnelles
Cet exercice pratique comporte une vulnérabilité d’injection SQL aveugle. L’application utilise un cookie de suivi à des fins d’analyse et exécute une requête SQL contenant la valeur du cookie soumis.

Les résultats de la requête SQL ne sont pas renvoyés, et l’application ne réagit pas différemment selon que la requête renvoie ou non des lignes. Si la requête SQL provoque une erreur, l’application renvoie alors un message d’erreur personnalisé.

La base de données contient une table appelée « users », comportant des colonnes intitulées « username » et « password ». Vous devez exploiter la vulnérabilité d’injection SQL aveugle pour découvrir le mot de passe de l’utilisateur administrateur.

Pour résoudre cet exercice, connectez-vous en tant qu’utilisateur administrateur.
#### Solution
**1. Confirmer l’injection**
- Visitez la page d’accueil de la boutique et utilisez Burp Suite pour intercepter et modifier la requête contenant le cookie `TrackingId`. Pour simplifier, la valeur d’origine est `TrackingId=xyz`.
- Ajoutez une apostrophe : `TrackingId=xyz'`. Vérifiez qu’un message d’erreur apparaît.
- Mettez maintenant deux apostrophes : `TrackingId=xyz''`. Vérifiez que l’erreur disparaît. Cela suggère qu’une erreur de syntaxe (ici, l’apostrophe non fermée) a un effet détectable sur la réponse.
- Il faut ensuite confirmer que le serveur interprète l’injection comme une requête SQL, et que l’erreur est bien une erreur de syntaxe SQL et non un autre type d’erreur. Pour cela, construisez d’abord une sous-requête en SQL valide : `TrackingId=xyz'||(SELECT '')||'`
- La requête semble toujours invalide. Cela peut venir du type de base de données. Essayez avec un nom de table prévisible : `TrackingId=xyz'||(SELECT '' FROM dual)||'`
- Comme l’erreur a disparu, la cible utilise probablement une base **Oracle**, qui exige que tout `SELECT` précise explicitement une table.
- Testez ensuite une requête invalide tout en gardant une syntaxe SQL valide, par exemple une table inexistante : `TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'`
- Cette fois, une erreur est renvoyée. Cela indique fortement que l’injection est traitée comme une requête SQL par le back-end.

**2. Déduire des informations avec les erreurs**
- Tant que vous injectez des requêtes syntaxiquement valides, la réponse d’erreur permet de déduire des informations sur la base.
- Pour vérifier que la table `users` existe : `TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'`
- Pas d’erreur, donc la table existe. La condition `WHERE ROWNUM = 1` est importante : elle évite de renvoyer plus d’une ligne, ce qui casserait la concaténation.

**3. Tester des conditions**
- Envoyez : `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`. Une erreur est reçue.
- Puis : `TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`. L’erreur disparaît.
- On peut donc déclencher une erreur conditionnellement à la vérité d’une condition. L’instruction `CASE` teste une condition et s’évalue à une expression si elle est vraie, à une autre si elle est fausse. La première contient une division par zéro, qui provoque l’erreur. Ici, les deux payloads testent `1=1` et `1=2`, et l’erreur survient quand la condition est vraie.

**4. Vérifier l’existence de l’utilisateur**
- `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
- Vérifiez que la condition est vraie (l’erreur est reçue) : l’utilisateur `administrator` existe.

**5. Déterminer la longueur du mot de passe**
- Envoyez : `TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
- Cette condition doit être vraie : le mot de passe fait plus d’un caractère.
- Testez ensuite `>2`, puis `>3`, et ainsi de suite. Vous pouvez le faire manuellement avec Burp Repeater, la longueur étant probablement courte.
- Quand la condition cesse d’être vraie (l’erreur disparaît), vous avez la longueur : **20 caractères**.
**6. Extraire chaque caractère avec Intruder**

- Il faut ensuite tester le caractère de chaque position. Cela demande beaucoup de requêtes, donc utilisez Burp Intruder (envoyez la requête via le menu contextuel).
- Dans Intruder, mettez ce cookie : `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
- `SUBSTR()` extrait un seul caractère du mot de passe pour le comparer à une valeur précise. L’attaque parcourt chaque valeur possible, une par une.
- Placez les marqueurs de payload autour du dernier `a` : sélectionnez uniquement ce `a` et cliquez sur « Add § ». Le cookie devient : `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
- Dans le panneau « Payloads », vérifiez que « Simple list » est sélectionné. Sous « Payload configuration », ajoutez les valeurs de `a` à `z` et de `0` à `9` (vous pouvez le faire avec « Add from list »). On suppose que le mot de passe ne contient que des minuscules et des chiffres.
- Lancez l’attaque avec « Start attack ».

**7. Lire les résultats**
- L’application renvoie un code HTTP **500** quand l’erreur se produit, et **200** normalement. La colonne « Status » permet de repérer facilement la ligne en 500. Le payload de cette ligne est le caractère à la première position.
- Relancez l’attaque pour chaque autre position : dans l’onglet Intruder d’origine, changez l’offset de `1` à `2`, soit `SUBSTR(password,2,1)='§a§'`.
- Lancez, relevez le caractère de la deuxième position, puis continuez avec les offsets 3, 4, etc., jusqu’à obtenir tout le mot de passe.

**8. Terminer le lab**
- Dans le navigateur, cliquez sur « My account » pour ouvrir la page de connexion et connectez-vous en tant que `administrator` avec le mot de passe trouvé.


## Liens
- [[Access-control]]
- [[Authentication]]
- [[Using-Burp-Suite]]
- [[Payloads-cheatsheet]]
