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
- 

## Pièges et points d'attention BSCP
- 

## Prévention
- 

## Labs PortSwigger
- [ ] Apprentice
- [ ] Practitioner
- [ ] Expert

## Journal des labs
- 

## Mes notes
- 

## Liens
- [[SSRF]]
- [[Path-traversal]]
- [[Information-disclosure]]
