---
tags: [bscp, server-side, sql-injection]
niveau: apprentice
statut: en cours
---
# Blind SQLi - Triggering time delay

## En bref


## Types et variantes



## Comment détecter


## Comment exploiter (principe)
- Déclencher un délai selon qu'une condition injectée est vraie ou fausse. Les requêtes SQL sont traitées de façon synchrone, donc un délai côté base retarde la réponse HTTP. Ce temps de réponse indique la valeur de vérité de la condition.
- Toutefois, la syntaxe du délai dépend du moteur de base de données.
```sql
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```
### Extraction de données caractère par caractère
```sql
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```
- Le délai se produit uniquement si la condition sur le caractère testé est vraie.
- On répète en fasant varier la position `SUBSTRING` et le caractère comparé pour reconstituer la valeur.

## Pièges et points d'attention BSCP


## Prévention


## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: Blind SQL injection with time delays and information retrieval
This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator` user.

To solve the lab, log in as the `administrator` user.
#### Solution
**Capturer la requête**

Je charge la page d'accueil du lab avec Burp en proxy, puis j'envoie la requête au Repeater. Le cookie TrackingId finit tel quel dans une requête SQL, c'est lui que je modifie. Rien ne s'affiche jamais, ni résultat ni erreur, donc je ne peux me fier qu'au temps de réponse.

**Confirmer l'injection**

J'ajoute `'||pg_sleep(5)--` à la fin de la valeur du cookie. La réponse met cinq secondes à revenir. L'injection marche, et pg_sleep me dit que c'est du PostgreSQL.

**Rendre le délai conditionnel**
Il faut que la pause dépende d'une condition, sinon elle ne m'apprend rien.

- **Condition vraie** : `'||(SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)||'` fait attendre cinq secondes.
- **Condition fausse** : avec `1=2` à la place, la réponse est immédiate.

J'ai maintenant un détecteur vrai ou faux, et tout le reste repose dessus.

**Vérifier que le compte existe**

`'||(SELECT CASE WHEN (username='administrator') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')||'`

Cinq secondes de pause, donc administrator est bien dans la table users.

**Trouver la longueur du mot de passe**

Avant de chercher les caractères, je cherche la longueur, ça évite de tirer dans le vide. Je teste `LENGTH(password)>1, puis >2`, et ainsi de suite, dans la même requête que ci-dessus. Tant que la réponse est lente, le mot de passe est plus long que le nombre testé. Le premier nombre sans délai donne la longueur : vingt caractères.

**Extraire le mot de passe**

La requête de base teste une seule position :

`'||(SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')||'`

À la main, ça ferait vingt positions fois trente-six caractères possibles. Je passe donc par l'Intruder.

- Type d'attaque : **Cluster bomb**.
- Marqueurs : un sur la position et un sur le caractère, ce qui donne `SUBSTRING(password,§1§,1)='§a§'.`
- Premier jeu de charges : les nombres de 1 à 20.
- Second jeu de charges : les lettres minuscules et les chiffres, car le mot de passe du lab n'a pas d'autre alphabet.
- Groupe de ressources : une seule requête à la fois. Si l'Intruder en envoie plusieurs en parallèle, les pauses se mélangent et les temps ne veulent plus rien dire.

J'affiche la colonne **Response received** et je trie dessus. Les lignes à cinq secondes ou plus sont les bonnes. Si la latence du lab crée des faux positifs, je passe à `pg_sleep(10)` et je relance.
Je relève un caractère par position, je les remets dans l'ordre et j'obtiens le mot de passe de l'administrator

**Se connecter**
Sur /login, je saisis administrator et ce mot de passe. Le lab est validé.

## Liens
- [[SQL-injection-Union-Attacks]]
- [[Blind-SQL-injection]]
- [[Error-based SQL Injection]]
- [[Payloads-cheatsheet]]
