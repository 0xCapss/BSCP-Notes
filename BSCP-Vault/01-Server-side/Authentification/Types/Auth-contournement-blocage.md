---
tags: [bscp, server-side, authentication]
niveau: practitioner
statut: à faire
---
# Authentification - contournement du verrouillage de compte et du blocage par IP

## En bref
- Les protections anti brute force (verrouillage du compte, blocage de l'IP) ont des limites que l'on peut contourner.
- Le seuil peut être scopé par IP, par compte ou par la combinaison des deux : cela change toute la stratégie.
- Note parente : [[Authentication]]

## Comment détecter
- Provoquer volontairement quelques échecs pour observer le déclencheur (nombre de tentatives, message affiché, durée).
- Vérifier si un login réussi remet le compteur à zéro.

## Comment exploiter (principe)
- Établir une liste de noms d'utilisateur potentiellement valides et une liste très restreinte de mots de passe probables.
- Avec Burp Intruder, tester chaque mot de passe contre chaque username (attaque en grille) : il suffit qu'un seul utilisateur ait choisi l'un des mots de passe testés pour compromettre un compte, sans jamais dépasser le seuil de verrouillage par compte.
- Le verrouillage par compte ne protège pas non plus contre le credential stuffing (test d'un grand nombre de paires username/password déjà connues, issues de fuites d'autres sites), qui exploite la réutilisation de mots de passe entre sites.

**Contournement d'un blocage par IP**
- Une IP peut être bloquée après un nombre trop élevé de tentatives, avec déblocage automatique après un délai, manuel par un administrateur, ou via un CAPTCHA résolu par l'utilisateur.
- Un attaquant peut manipuler son IP apparente (en-têtes `X-Forwarded-For`, `X-Real-IP`, etc.) pour contourner ce blocage si l'application fait confiance à un en-tête fourni par le client plutôt qu'à la connexion réelle (voir [[Payloads-cheatsheet]]).

**Plusieurs identifiants dans une seule requête**
- Si l'application accepte un tableau JSON pour le champ mot de passe (`"password": ["123456", "password", ...]`), tous les candidats peuvent être testés dans une seule requête, ce qui contourne une limitation par nombre de requêtes.

## Pièges et points d'attention BSCP
- Inclure ses propres identifiants valides à intervalles réguliers dans la liste testée suffit souvent à passer sous le radar si le compteur se réinitialise après un login réussi.
- Avec `X-Forwarded-For`, utiliser une valeur différente à chaque requête (payload Numbers ou script Turbo Intruder, voir [[Payloads-cheatsheet]]).
- Burp Intruder : limiter à une seule requête simultanée (resource pool) quand l'ordre des essais compte.

## Labs PortSwigger
- [[Authentication#Lab 4 - Protection anti brute force défaillante, blocage par IP]] (Practitioner)
- [[Authentication#Lab 6 - Protection anti brute force défaillante, plusieurs identifiants par requête]] (Expert)

## Liens
- [[Authentication]]
- [[Auth-brute-force-identifiants]]
