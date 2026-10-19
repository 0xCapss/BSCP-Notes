---
tags: [bscp, server-side, command-injection]
niveau: apprentice
statut: à faire
---
# OS command injection

## En bref
- Elle permet à un attaquant d'exécuter des commandes du systèmes d'exploitation sur le serveur web.
- Généralement, elle permet de compromettre l'entièreté de l'application.
- Par extension, l'attaquant peut souvent s'en servir pour compromettre d'autres parties de l'infrastructure d'hébergement.
## Commandes utiles
| Objectif                     | Linux         | Windows         |
| ---------------------------- | ------------- | --------------- |
| Nom de l'utilisateur courant | `whoami`      | `whoami`        |
| Système d'exploitation       | `uname -a`    | `ver`           |
| Configuration réseau         | `ifconfig`    | `ipconfig /all` |
| Connexions réseau            | `netstat -an` | `netstat -an`   |
| Processus en cours           | `ps -ef`      | `tasklist`      |
- Ces commandes servent à la reconnaissance une fois l'injection confirmée : identifier le compte sous lequel tourne l'application, le système, le réseau...
- Toutefois sur Linux, `ifconfig` peut être absent des systèmes récents. `ip a` est alors l'équivalent.

## Types et variantes
- 

## Comment détecter
- 

## Comment exploiter (principe)
### Injection de commande
- Une application permet de vérifier si un article est en stock dans un magasin donné, via l'URL `https://insecure-website.com/stockStatus?productID=381&storeID=29`.
- Pour fournir ces informations, elle interroge des systèmes hérités.
- a fonctionnalité appelle une commande shell avec les identifiants du produit et du magasin en arguments : `stockreport.pl 381 29`.
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

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
### Lab: OS command injection, simple case
This lab contains an OS command injection vulnerability in the product stock checker.
The application executes a shell command containing user-supplied product and store IDs, and returns the raw output from the command in its response.
To solve the lab, execute the `whoami` command to determine the name of the current user.

## Mes notes
- 

## Liens
- [[SSRF]]
- [[Path-traversal]]
- [[Information-disclosure]]
